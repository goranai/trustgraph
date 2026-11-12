---
layout: default
title: "Ontology Service Technical Specification"
parent: "Tech Specs"
---

# Ontology Service Technical Specification

## Problem Statement

Ontology handling in TrustGraph has three problems.

### Non-standard ontology format at the API boundary

Ontologies are loaded as a custom JSON structure with keys like
`classes`, `objectProperties`, and `datatypeProperties`. This format
maps closely to OWL concepts but is not a recognised standard. Anyone
loading an ontology must convert from whatever format they have (Turtle,
OWL/XML, JSON-LD, Protégé export) into this bespoke schema. There is no
tooling ecosystem for it, no validators, and no documentation outside
this project.

Standard serialisations already exist for exactly this data: JSON-LD
and Turtle are both W3C standards, widely supported by tools like
rdflib, and can represent the same OWL classes, properties, domains,
ranges, and hierarchies that the custom format carries. Accepting a
standard format at the API boundary would eliminate the conversion step
and let users work with familiar tooling.

### Ontology management is embedded in the extract processor

The ontology extract processor (`extract.py`, ~1,090 lines) directly
owns ontology loading, embedding, subset selection, URI expansion,
triple validation, and ontology triple generation. The query side has
its own parallel implementation (`query/ontology/`) that was scaffolded
but never completed or integrated.

This coupling means:

- Ontology logic cannot be reused by other consumers (e.g. a
  NL-to-SPARQL query service) without duplicating it.
- The extract processor is responsible for both text extraction and
  ontology management, making it harder to reason about and test.
- Changes to ontology handling (format support, validation rules,
  embedding strategy) require modifying the extract processor.

A separate ontology service would own parsing, validation, storage,
embedding, and subset selection, exposing these capabilities to any
consumer that needs ontology context.

### No natural-language-to-SPARQL query path

TrustGraph has a SPARQL query service that evaluates SPARQL against the
knowledge graph, but there is no way to get from a natural language
question to a SPARQL query. The scaffolded `query/ontology/` package
includes a `SPARQLGenerator` but it was never tested or integrated — its
host service (`OntoRAGQueryService`) does not even start.

A key obstacle was entity resolution: translating a natural language
mention like "Apple" into the correct knowledge graph URI. Without that,
generated SPARQL can only use string matching or hope that entity names
match graph identifiers exactly.

Typed graph embeddings now solve this. Entity embeddings stored in
Qdrant carry `rdf_type` metadata, so a semantic search for "Apple"
filtered to type `Organization` returns the correct entity URI. This
makes a viable NL-to-SPARQL path possible:

1. Use the ontology schema to help an LLM identify entities, their
   likely types, and the relationships being asked about.
2. Resolve each entity mention to a knowledge graph URI via typed
   graph embedding lookup.
3. Construct SPARQL using the resolved URIs and ontology properties.

This path requires ontology context (the schema) to be available as a
service, reinforcing the need for the ontology service described above.

## Goals

- **Standard ontology formats**: accept JSON-LD and Turtle at the API
  boundary, parsed via rdflib. Retire the custom JSON format.
- **Ontology service**: a separate service that owns ontology parsing,
  validation, embedding, and subset selection, available to any
  consumer over pub/sub.
- **NL-to-SPARQL query path**: a service that translates natural
  language questions to SPARQL queries, using ontology context for
  schema awareness and typed graph embeddings for entity resolution.
- **Extract processor simplification**: the ontology extract processor
  becomes a client of the ontology service rather than owning ontology
  management directly.
- **No breaking changes to extraction output**: the knowledge graph
  triples and entity contexts produced by extraction remain the same.

### Future goals

- **Ontology-guided GraphRAG**: use ontology context to automatically
  derive guided traversal constraints (hop types, relationship filters)
  from a natural language question, so that GraphRAG queries benefit
  from schema awareness without the caller specifying guidance manually.

## Design Decisions

- **JSON-LD as internal canonical format**: ontologies are stored and
  transmitted as JSON-LD. Turtle is accepted at the ingestion boundary
  and converted on arrival — a lossless, trivial conversion via rdflib.
- **rdflib for parsing and validation**: already a dependency; handles
  format conversion and gives OWL/RDFS validation for free.
- **Pub/sub FlowProcessor pattern**: the ontology service and
  NL-to-SPARQL service follow the standard FlowProcessor pattern,
  consistent with the rest of the architecture.
- **LLM-based SPARQL generation**: the LLM receives ontology schema
  context and resolved entity URIs. The scaffolded template approach
  was too brittle for real use.
- **Entity resolution via existing typed graph embeddings**: reuse the
  Qdrant infrastructure with `rdf_type` filtering rather than building
  a parallel entity resolver.
- **Retire the `query/ontology/` scaffold**: replace entirely rather
  than retrofit; it was never tested or integrated.
- **Python API SDK and CLI support**: new services get SDK bindings and
  CLI utilities so that users can interact with the ontology service
  and NL-to-SPARQL service directly for experimentation and debugging.

## Services

### New services

- **Ontology service**: owns ingestion (JSON-LD/Turtle parsing),
  validation, embedding, and subset selection. Serves ontology context
  to any consumer over pub/sub.
- **NL-to-SPARQL service**: takes a natural language question, calls
  the ontology service for schema context, resolves entities via typed
  graph embeddings, uses an LLM to generate SPARQL, and passes the
  result to the existing SPARQL query service for execution.

### Modified services

- **Ontology extract processor**: simplified to be a client of the
  ontology service for subset selection and validation, rather than
  owning that logic itself.

### Retired

- **`query/ontology/` package**: replaced by the NL-to-SPARQL service.
- **Custom JSON ontology format**: replaced by JSON-LD/Turtle.

## Open Design Questions

### Extraction during ontology loading

The ontology service may take time to parse, validate, and embed a
large ontology. Extraction consumers need a strategy for handling
messages that arrive before the ontology is ready. Options include
blocking until ready, rejecting messages with a retriable error, or
extracting without ontology context and re-processing later. This
needs a decision.

### External ontology references

OWL ontologies can reference external ontologies via `owl:imports`.
We need to decide whether the ontology service resolves these
transitively (fetching and merging imported ontologies), rejects
ontologies with external references, or ignores the imports and works
only with what is explicitly provided. Each has different implications
for reproducibility and network dependencies.

### Ontology loading lifecycle

Ontologies should load in response to configuration events, not on
first processing message. This keeps the loading lifecycle predictable
and avoids latency surprises on the first extraction request. The
ontology service should subscribe to config updates, parse and embed
on receipt, and signal readiness to consumers.

### Ontology matching strategy

The current subset selection uses embedding similarity to match text
to ontology elements. This could be improved — for instance by adding
a reranking stage after initial retrieval, or by making the matching
strategy configurable (pure embedding, embedding + rerank, LLM-based
selection). The ontology service should be designed to allow the
matching approach to evolve without changing the consumer interface.
