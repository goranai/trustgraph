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
