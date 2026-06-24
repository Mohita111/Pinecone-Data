> ## Documentation Index
> Fetch the complete documentation index at: https://docs.pinecone.io/llms.txt
> Use this file to discover all available pages before exploring further.

# Concepts

> Understand concepts in Pinecone and how they relate to each other.

<img className="block max-w-full dark:hidden" noZoom src="https://mintcdn.com/pinecone/pBbXhyz41iPzpfh0/images/objects.png?fit=max&auto=format&n=pBbXhyz41iPzpfh0&q=85&s=6aaa656d58401fc1cdd75974ab1d1c73" width="1180" height="600" data-path="images/objects.png" />

<img className="hidden max-w-full dark:block" noZoom src="https://mintcdn.com/pinecone/pBbXhyz41iPzpfh0/images/objects-dark.png?fit=max&auto=format&n=pBbXhyz41iPzpfh0&q=85&s=5f70c26d69d475be702101cfc2c5a307" width="1180" height="600" data-path="images/objects-dark.png" />

## Organization

An organization is a group of one or more [projects](#project) that use the same billing. Organizations allow one or more [users](#user) to control billing and permissions for all of the projects belonging to the organization.

For more information, see [Understanding organizations](/guides/organizations/understanding-organizations).

## Project

A project belongs to an [organization](#organization) and contains one or more [indexes](#index). Each project belongs to exactly one organization, but only [users](#user) who belong to the project can access the indexes in that project. [API keys](#api-key) and [Assistants](#assistant) are project-specific.

For more information, see [Understanding projects](/guides/projects/understanding-projects).

## Index

Pinecone [serverless indexes](/guides/index-data/indexing-overview) hold your data as [documents](#document) — JSON objects with ranking fields that Pinecone indexes according to a schema you define, plus any number of metadata fields. A single index can mix multiple ranking field types: a `dense_vector` field for [semantic search](#index-with-dense-vectors), a `sparse_vector` field for [sparse-vector retrieval](#index-with-sparse-vectors), and one or more `string` fields with `full_text_search` enabled for [full-text search](#full-text-search) with BM25 and Lucene queries. Metadata fields (anything else you upsert) are auto-indexed for filtering at upsert time — no schema declaration required.

One index per use case is the typical pattern. Because a document can combine vectors, text, and metadata in the same record, a single index often covers what previously required two — pick the ranking signal per query with `score_by`.

### Full-text search

Full-text search is **BM25 token matching with Lucene query syntax** over text fields in your schema — `string` fields you've declared with `full_text_search` so their content is indexed for token-level retrieval. "Text field" is the colloquial name; the JSON `type` is `string`. No model required — Pinecone handles tokenization, IDF, and length normalization at index time and BM25 scoring at query time. "Token" here means a unit produced by Pinecone's text analyzer (whitespace + punctuation split, lowercased, optionally stemmed) — not the subword unit a dense or sparse embedding model uses internally. See [Tokens and analyzers](/guides/search/full-text-search#tokens-and-analyzers) for the full pipeline.

How it works:

1. You upsert data as JSON [documents](#document).
2. You declare each ranking field's type in the index schema: `dense_vector`, `sparse_vector`, or `string` with `full_text_search` (indexed for BM25 ranking and Lucene queries). Metadata fields are not declared in the schema.
3. Pinecone indexes each ranking field according to its declared type and auto-indexes any other fields on the document for metadata filtering.

When you search, you choose a scoring method via `score_by`. The literal value of `type` selects the method: `text` (BM25 token matching on a single text field), `query_string` ([Lucene query syntax](/guides/search/full-text-search#query-syntax-reference) across one or more text fields, including cross-field boolean queries), `dense_vector` (vector similarity), or `sparse_vector` (sparse-vector similarity). Any scoring method can be combined with metadata filters — including logical operators (`$and`, `$or`, `$not`), existence checks (`$exists`), and the text-match operators (`$match_phrase`, `$match_all`, `$match_any`) for phrase and token matching against text fields.

Use full-text search for keyword and phrase search over text content — product names, identifiers, technical terms, code, and other cases where queries and documents share specific tokens. For sparse-vector retrieval with a learned encoder (such as [`pinecone-sparse-english-v0`](/models/pinecone-sparse-english-v0)), see [Index with sparse vectors](#index-with-sparse-vectors). For semantic similarity over natural-language queries, see [Index with dense vectors](#index-with-dense-vectors).

Learn more:

* [Full-text search](/guides/search/full-text-search)
* [Document](#document)

### Index with dense vectors

These indexes store records that each have one [dense vector](#dense-vector). A dense vector is a series of numbers that represent the meaning and relationships of text, images, or other data. Each vector is a point in a multidimensional space; each number is a coordinate in that space. Vectors that are closer together in that space are semantically similar.

When you query an index with dense vectors, Pinecone retrieves records whose vectors are most semantically similar to the query. This is often called **semantic search**, nearest neighbor search, similarity search, or just vector search.

<Note>
  If records in an index with dense vectors also have a [sparse vector](#sparse-vector), the index supports single-index [hybrid search](/guides/search/hybrid-search#use-a-single-index-for-dense-and-sparse-vectors) on the same records. This single-index pattern uses the vector API and isn't available for indexes with document schemas. To combine a lexical signal with a dense signal in an index with a document schema, restrict a dense search with a text-match filter or run separate searches and merge the results client-side; see [Hybrid search](/guides/search/hybrid-search).
</Note>

### Index with sparse vectors

These indexes store records that each have one [sparse vector](#sparse-vector) — a vector with very high dimensionality but only a small number of non-zero values. Each dimension typically corresponds to a token in a vocabulary; the non-zero values represent the importance of those tokens in a document.

When you search an index with sparse vectors, Pinecone retrieves records whose vectors share the most weighted tokens with the query vector. This is often called **sparse-vector retrieval** or **sparse-vector lexical search**.

Sparse vectors are produced by a sparse embedding model. Pinecone hosts [`pinecone-sparse-english-v0`](/models/pinecone-sparse-english-v0), a learned-sparse encoder that predicts per-token weights and includes term expansion (related concepts that don't appear in the source text). You can also bring your own sparse model.

<Note>
  **Sparse-vector lexical search vs. full-text search.** Both retrieve documents using token-level signals over an inverted index. They differ in how tokens are weighted: [full-text search](#full-text-search) uses **BM25** — a statistical scoring function with no machine learning, computed at query time over your raw text fields. Sparse-vector lexical search uses a **learned sparse encoder** that produces token weights at index time, often with term expansion. Use full-text search when you want a strong baseline with no model to manage; use sparse vectors when a learned encoder (yours or Pinecone's hosted one) better captures your domain's term importance and synonyms.

  A useful gradient: dense ranks on **concept** (semantic similarity), full-text search ranks on **strict character-level token matching** (BM25), and sparse-vector lexical search sits **between them** — token-aware, but with learned per-token weights and term expansion. Sparse vectors carry no positional information, so phrase matching (`"machine learning"` as a contiguous span) requires full-text search, not sparse.
</Note>

## Namespace

A namespace is a partition within an index. It divides [records](#record) into separate groups so that each query scans only one namespace (faster lookups) and each customer's data can be isolated from another customer's (multitenant isolation).

All [upserts](/guides/index-data/upsert-data), [queries](/guides/search/search-overview), and other data read and write operations always target one namespace:

<img className="block max-w-full dark:hidden" noZoom src="https://mintcdn.com/pinecone/pBbXhyz41iPzpfh0/images/quickstart-upsert.png?fit=max&auto=format&n=pBbXhyz41iPzpfh0&q=85&s=c852ca55745a70ed51e5892330751830" width="1400" height="783" data-path="images/quickstart-upsert.png" />

<img className="hidden max-w-full dark:block" noZoom src="https://mintcdn.com/pinecone/pBbXhyz41iPzpfh0/images/quickstart-upsert-dark.png?fit=max&auto=format&n=pBbXhyz41iPzpfh0&q=85&s=93bc6a0986fab5c8a736ca0bb50834bd" width="1400" height="783" data-path="images/quickstart-upsert-dark.png" />

For more information, see [Use namespaces](/guides/index-data/indexing-overview#namespaces).

## Record

A record is the unit of data for [indexes with dense vectors](#index-with-dense-vectors) and [indexes with sparse vectors](#index-with-sparse-vectors): a [record ID](#record-id), one vector (or both vector types for single-index [hybrid search](/guides/search/hybrid-search)), and optional [metadata](#metadata). With [integrated embedding](/guides/index-data/indexing-overview#integrated-embedding) you can upsert raw text instead of a vector, and Pinecone embeds it at index time. When an item has more than one searchable field — say, a text field ranked by BM25 alongside a `dense_vector` field for similarity — model it as a [document](#document).

For more information, see [Upsert data](/guides/index-data/upsert-data).

## Document

A document is the unit of data in an index with a document schema — a JSON object with a required `_id` field, the ranking fields declared in the index's schema, and any number of metadata fields. Documents support multiple ranking field types in a single record: a `dense_vector` field (for [semantic search](#index-with-dense-vectors)), a `sparse_vector` field (for [sparse-vector retrieval](#index-with-sparse-vectors)), and one or more `string` fields with `full_text_search` enabled (for [full-text search](#full-text-search)). A single document can carry vectors, text, and metadata together, and you choose the scoring method per query via `score_by`. Documents are the recommended shape for new multi-field and full-text workloads; vector-only indexes continue to use [records](#record). Both APIs are fully supported.

Document fields can hold structured values: a metadata `string_list` field holds an array of strings; a `dense_vector` field holds an array of floats; a `sparse_vector` field is an object with two parallel arrays — `indices` (token positions) and `values` (token weights).

A schema can declare up to 100 `string` fields with `full_text_search` enabled, but at most one `dense_vector` field and at most one `sparse_vector` field per index.

Metadata fields are not declared in the schema. Any field on an upserted document that is not declared in the schema is stored, returned via `include_fields`, and automatically indexed for filtering. Pinecone infers metadata field types (string, number, boolean, array of strings) from the values you upsert.

Field names must be unique, non-empty strings, must not start with `_` (reserved for system-managed fields like `_id` and `_score`) or `$` (reserved for filter operators), and are limited to 64 bytes.

For more information, see [Full-text search](/guides/search/full-text-search).

### Record ID

A record ID is a record's unique ID. [Use ID prefixes](/guides/index-data/data-modeling#use-structured-ids) that reflect the type of data you're storing.

### Dense vector

A dense vector, also referred to as a vector embedding or simply a vector, is a series of numbers that represent the meaning and relationships of data. Each vector is a point in a multidimensional space; each number is a coordinate in that space. Vectors that are closer together in that space are semantically similar.

Dense vectors are stored in indexes (see [Index with dense vectors](#index-with-dense-vectors)).

You use a dense embedding model to convert data to dense vectors. The embedding model can be external to Pinecone or [hosted on Pinecone infrastructure](/guides/index-data/create-an-index#embedding-models) and integrated with an index.

For more information about dense vectors, see [What are vector embeddings?](https://www.pinecone.io/learn/vector-embeddings/).

### Sparse vector

Sparse vectors are often used to represent documents or queries in a way that captures keyword information. Each dimension in a sparse vector typically represents a word from a dictionary, and the non-zero values represent the importance of these words in the document.

Sparse vectors have a large number of dimensions, but a small number of those values are non-zero. Because most values are zero, Pinecone stores sparse vectors efficiently by keeping only the non-zero values along with their corresponding indices.

Sparse vectors are stored in indexes (see [Index with sparse vectors](#index-with-sparse-vectors)) and can also coexist with dense vectors in a single index for [hybrid search](/guides/search/hybrid-search#use-a-single-index-for-dense-and-sparse-vectors) on the vector API. To combine a lexical signal with a dense signal in an index with a document schema, restrict a dense search with a text-match filter on a `string` field with `full_text_search` enabled or run separate searches and merge the results client-side. To convert data to sparse vectors, use a sparse embedding model. The embedding model can be external to Pinecone or [hosted on Pinecone infrastructure](/guides/index-data/create-an-index#embedding-models) and integrated with an index.

For more information about sparse vectors, see [Sparse retrieval](https://www.pinecone.io/learn/sparse-retrieval/).

### Metadata

Metadata is additional information included in a record to provide more context and enable additional [filtering capabilities](/guides/index-data/indexing-overview#metadata). For example, the original text that was embedded can be stored in the metadata.

## Other concepts

Although not represented in the diagram above, Pinecone also contains the following concepts:

* [API key](#api-key)
* [User](#user)
* [Backup or collection](#backup-or-collection)
* [Pinecone Inference](#pinecone-inference)
* [Read unit (RU)](#read-unit-ru)
* [Write unit (WU)](#write-unit-wu)

### API key

An API key is a unique token that [authenticates](/reference/api/authentication) and authorizes access to the [Pinecone APIs](/reference/api/introduction). API keys are project-specific.

### User

A user is a member of organizations and projects. Users are assigned specific roles at the organization and project levels that determine the user's permissions in the [Pinecone console](https://app.pinecone.io).

For more information, see [Manage organization members](/guides/organizations/manage-organization-members) and [Manage project members](/guides/projects/manage-project-members).

### Backup or collection

A backup is a static copy of a serverless index.

Backups only consume storage. They are non-queryable representations of a set of records. You can create a backup from an index, and you can create a new index from that backup. The new index configuration can differ from the original source index: for example, it can have a different name. However, it must have the same number of dimensions and similarity metric as the source index.

For more information, see [Understanding backups](/guides/manage-data/backups-overview).

### Pinecone Inference

Pinecone Inference is an API service that provides access to [embedding models](/guides/index-data/create-an-index#embedding-models) and [reranking models](/guides/search/rerank-results#reranking-models) hosted on Pinecone's infrastructure.

### Read unit (RU)

A read unit (RU) is the unit Pinecone uses to measure and price the cost of read requests to a serverless index, including [query](/guides/manage-cost/understanding-cost#query), [fetch](/guides/manage-cost/understanding-cost#fetch), and [list](/guides/manage-cost/understanding-cost#list).

For how read units are calculated, see [Understanding cost](/guides/manage-cost/understanding-cost#read-units).

### Write unit (WU)

A write unit (WU) is the unit Pinecone uses to measure and price the cost of write requests to a serverless index, including [upsert](/guides/manage-cost/understanding-cost#upsert), [update](/guides/manage-cost/understanding-cost#update), and [delete](/guides/manage-cost/understanding-cost#delete).

For how write units are calculated, see [Understanding cost](/guides/manage-cost/understanding-cost#write-units).

## Learn more

* [Vector database](https://www.pinecone.io/learn/vector-database/)
* [Pinecone APIs](/reference/api/introduction)
* [Approximate nearest neighbor (ANN) algorithms](https://www.pinecone.io/learn/a-developers-guide-to-ann-algorithms/)
* [Retrieval augmented generation (RAG)](https://www.pinecone.io/learn/retrieval-augmented-generation/)
* [Image search](https://www.pinecone.io/learn/series/image-search/)
* [Tokenization](https://www.pinecone.io/learn/tokenization/)
