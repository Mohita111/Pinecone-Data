> ## Documentation Index
> Fetch the complete documentation index at: https://docs.pinecone.io/llms.txt
> Use this file to discover all available pages before exploring further.

# Data modeling

> Learn how to structure records for efficient data retrieval and management in Pinecone.

## Documents

A document is the unit of data in an index with a document schema — a JSON object with a required `_id` field, the ranking fields declared in the index's schema, and any number of metadata fields. Documents support multiple field types in a single record: a `dense_vector` field (for [semantic search](/guides/search/semantic-search)), a `sparse_vector` field (for [sparse-vector lexical search](/guides/search/lexical-search)), one or more `string` fields with `full_text_search` enabled (for [full-text search](/guides/search/full-text-search) with BM25 and Lucene queries), plus any metadata you upsert alongside them.

The schema, declared at index creation, tells Pinecone how to rank each ranking field. Schema field types:

* `dense_vector` — indexed for ANN similarity search.
* `sparse_vector` — indexed for sparse-vector lexical search.
* `string` with a nested `full_text_search` config object (`{}` enables with all defaults; optional sub-fields: `language`, `stemming`, `stop_words`) — indexed for **BM25** ranking and Lucene queries. Lowercasing and the token length cap are server-applied and cannot be overridden.

Metadata fields are not declared in the schema. Any field you upsert that is not declared in the schema is stored on the document, returned via `include_fields`, and automatically indexed for filtering. Pinecone infers the metadata field type from the values you upsert: strings, numbers (floating point), booleans, and arrays of strings are all supported.

Document fields can hold structured values: a metadata `string_list` field holds an array of strings; a `dense_vector` field holds an array of floats; a `sparse_vector` field is an object with two parallel arrays — `indices` (token positions) and `values` (token weights).

A schema can declare up to 100 `string` fields with `full_text_search` enabled, but at most one `dense_vector` field and at most one `sparse_vector` field per index.

Example document for an index with `title`, `body`, `embedding`, and `category` fields:

```json theme={null}
{
  "_id": "document1#chunk1",
  "title": "Introduction to Vector Databases",
  "body": "First chunk of the document content...",
  "embedding": [0.0236, -0.0329, ..., -0.0104, 0.0086],
  "category": "tutorial"
}
```

Field-name rules:

* Must be unique, non-empty strings.
* Must not start with `_` (reserved for system-managed fields like `_id` and `_score`) or `$` (reserved for filter operators).
* Limited to 64 bytes.

For the full schema reference (language and analyzer options, multi-field schemas, scoring methods), see [Full-text search](/guides/search/full-text-search).

<Note>
  **Chunking granularity.** A document is the unit of retrieval — `top_k` and `_score` are computed per document, not per sub-section. In public preview, Pinecone does not split a single document into multiple in-document chunks at index time. If your source content is longer than what you want to retrieve as one hit (a long article, a PDF, a transcript), do the chunking in your application before upsert and store each chunk as its own document, with an ID like `document1#chunk1`, `document1#chunk2`, and a metadata field that ties chunks back to the parent document for grouping at query time.
</Note>

### Schema patterns

The same document model supports several common schema shapes. Pick the one that matches the signal you want to rank by, and plan your fields up front: in public preview, schema migration is not supported after index creation. Filters are deterministic per document and apply before scoring; choose your hard yes/no constraints (including text-match operators on FTS-enabled `string` fields) first, then pick a `score_by` method to rank whatever remains. See [Filters vs. scoring](/guides/search/full-text-search#filters-vs-scoring).

<Note>
  The Python snippets in each accordion below assume an initialized client and the schema-builder import:

  ```python Python theme={null}
  from pinecone import Pinecone
  from pinecone.preview import SchemaBuilder

  pc = Pinecone(api_key="YOUR_API_KEY")
  ```

  Each accordion shows the pattern-specific schema, an example document, and a search snippet. The control-plane (`pc.preview.indexes.create(...)`) and data-plane (`index = pc.preview.index(name=...)`) calls in the snippets reuse this `pc`.
</Note>

<AccordionGroup>
  <Accordion title="Single text field — keyword search only (FTS)">
    Use when you want BM25 keyword ranking on one piece of text per document (a review body, a support ticket, a product description) and you don't have embeddings to manage.

    ```python Python theme={null}
    from pinecone.preview import SchemaBuilder

    schema = (
        SchemaBuilder()
        .add_string_field("review_text", full_text_search={"language": "en"})
        .build()
    )

    pc.preview.indexes.create(name="book-reviews", schema=schema)
    ```

    A document upserted into this index looks like:

    ```json theme={null}
    {
      "_id": "review-1234",
      "review_text": "Beautifully written exploration of contact, communication, and civilization across cosmic distances. The pacing is uneven but the central premise carries you through."
    }
    ```

    Search with a single `text` clause (the score\_by `type`, not a field type — this clause runs BM25 ranking on the named string field):

    ```python Python theme={null}
    index.documents.search(
        namespace="reviews",
        top_k=10,
        score_by=[{"type": "text", "field": "review_text", "query": "civilization"}],
    )
    ```

    See [Full-text search](/guides/search/full-text-search).
  </Accordion>

  <Accordion title="Multi-field FTS — score across two text fields (e.g. body + summary)">
    Use when a document has more than one piece of text that should both contribute to ranking — for example, a long `review_text` plus a short `review_summary`. Pinecone combines the per-field BM25 scores into one ranking per document.

    ```python Python theme={null}
    schema = (
        SchemaBuilder()
        .add_string_field("review_text", full_text_search={"language": "en"})
        .add_string_field("review_summary", full_text_search={"language": "en"})
        .build()
    )

    pc.preview.indexes.create(name="book-reviews-multi", schema=schema)
    ```

    A document upserted into this index looks like:

    ```json theme={null}
    {
      "_id": "review-1234",
      "review_text": "Beautifully written exploration of contact, communication, and civilization across cosmic distances. The pacing is uneven but the central premise carries you through.",
      "review_summary": "Monumental science fiction with uneven pacing",
      "category": "science-fiction",
      "rating": 4.5
    }
    ```

    `category` and `rating` are not declared in the schema — they're upserted as metadata, automatically indexed for filtering, and usable in `filter` expressions.

    Pass two `text` clauses in `score_by`; the server combines them into one ranking, with each contributing field weighted equally in `2026-01.alpha`.

    ```python Python theme={null}
    index = pc.preview.index(name="book-reviews-multi")

    index.documents.search(
        namespace="reviews",
        top_k=5,
        score_by=[
            {"type": "text", "field": "review_text",    "query": "disappointing"},
            {"type": "text", "field": "review_summary", "query": "Disappointing"},
        ],
        include_fields=["*"],
    )
    ```
  </Accordion>

  <Accordion title="Dense + FTS — semantic and keyword in one index">
    Most workloads that combine semantic ranking with keyword matching reach for this pattern: rank by dense (or sparse) similarity, restricted to documents that contain a specific term or phrase. Common examples include semantic search over patents, regulatory filings, internal knowledge bases, or other technical literature where the right answer must contain a specific term. A single schema can include one `dense_vector` field plus any number of FTS-enabled string fields:

    ```python Python theme={null}
    schema = (
        SchemaBuilder()
        .add_string_field("book_title", full_text_search={"language": "en"})
        .add_string_field("review_text", full_text_search={"language": "en"})
        .add_dense_vector_field("review_embedding", dimension=1024, metric="cosine")
        .build()
    )

    pc.preview.indexes.create(name="book-reviews-dense", schema=schema)
    ```

    A document upserted into this index looks like:

    ```json theme={null}
    {
      "_id": "review-1234",
      "book_title": "The Three-Body Problem",
      "review_text": "Beautifully written exploration of contact, communication, and civilization across cosmic distances.",
      "review_embedding": [0.012, -0.087, 0.153, ...]
    }
    ```

    `review_embedding` is a 1024-dim list of floats produced by your dense embedding model. Use the same model at query time so the query vector lives in the same space.

    A single search request ranks by one scoring type. With this schema you have two query options:

    **Option A — dense ranking restricted by a text-match filter** (the most common hybrid pattern):

    ```python Python theme={null}
    index = pc.preview.index(name="book-reviews-dense")

    # query_embedding is a 1024-dim list of floats from your embedding model.
    query_embedding = embed("beautifully written, hard sci-fi")

    index.documents.search(
        namespace="reviews",
        top_k=5,
        score_by=[
            {"type": "dense_vector", "field": "review_embedding", "values": query_embedding},
        ],
        filter={"review_text": {"$match_phrase": "beautifully written"}},
    )
    ```

    **Option B — run BM25 and dense searches separately and merge client-side** (when you want both signals to contribute to ranking, e.g. via reciprocal rank fusion):

    ```python Python theme={null}
    dense_hits = index.documents.search(
        namespace="reviews", top_k=50,
        score_by=[{"type": "dense_vector", "field": "review_embedding", "values": query_embedding}],
    )
    bm25_hits = index.documents.search(
        namespace="reviews", top_k=50,
        score_by=[{"type": "text", "field": "review_text", "query": "beautifully written"}],
    )
    # Merge dense_hits + bm25_hits in your application (e.g. RRF) to produce final ranking.
    ```

    See [Hybrid search](/guides/search/hybrid-search) for a fuller discussion.

    <Tip>
      The `dense_vector` field's source content is independent of the FTS-enabled `string` fields it sits alongside. You can embed images (e.g., with a multimodal model like Gemini Embedding 2 or a CLIP-style model) and pair them with FTS-enabled `string` fields holding captions, geography, or taxonomy — then query the image vector with a text description and restrict matches with FTS filters on those `string` fields. The schema doesn't constrain what the dense vector represents; it just stores a vector of the declared dimension.
    </Tip>
  </Accordion>

  <Accordion title="Multi-signal index — dense + sparse + FTS in one schema">
    Use when a single document is best described by more than one ranking signal — for example, a video catalog where each item has frame embeddings (dense), auto-generated captions you've encoded as sparse vectors (sparse), and a transcript text field (BM25/Lucene). One schema declares all three; you pick the ranking signal per query with `score_by`. No second index, no cross-index linkage to maintain.

    ```python Python theme={null}
    schema = (
        SchemaBuilder()
        .add_dense_vector_field("frame_embedding", dimension=1024, metric="cosine")
        .add_sparse_vector_field("caption_sparse")
        .add_string_field("transcript", full_text_search={"language": "en"})
        .build()
    )

    pc.preview.indexes.create(name="video-catalog", schema=schema)
    ```

    A `language` field upserted alongside these ranking fields is treated as metadata: stored on the document, returned via `include_fields`, and auto-indexed for filtering.

    A document upserted into this index looks like:

    ```json theme={null}
    {
      "_id": "video-7890#scene-3",
      "frame_embedding": [0.012, -0.087, 0.153, ...],
      "caption_sparse": {
        "indices": [42, 1077, 9821],
        "values":  [0.41, 0.33, 0.18]
      },
      "transcript": "I think we should go now before it gets dark.",
      "language": "en"
    }
    ```

    `frame_embedding` is a 1024-dim list of floats from your dense vision model. `caption_sparse` is the output of your sparse encoder — an object with parallel `indices` (token IDs) and `values` (token weights) arrays.

    The same index supports three different query shapes. All three assume:

    ```python Python theme={null}
    index = pc.preview.index(name="video-catalog")

    # Replace with the outputs of your encoders.
    query_embedding = embed_image(query_image)                 # 1024-dim list of floats
    query_sparse    = sparse_encode("scene with a lighthouse") # {"indices": [...], "values": [...]}
    ```

    **Semantic frame search** — rank by visual similarity:

    ```python Python theme={null}
    index.documents.search(
        namespace="videos",
        top_k=10,
        score_by=[{"type": "dense_vector", "field": "frame_embedding", "values": query_embedding}],
    )
    ```

    **Caption lexical search** — rank by sparse-vector lexical similarity over your encoded captions:

    ```python Python theme={null}
    index.documents.search(
        namespace="videos",
        top_k=10,
        score_by=[{"type": "sparse_vector", "field": "caption_sparse", "sparse_values": query_sparse}],
    )
    ```

    **Semantic search restricted to spoken phrase** — semantic frame ranking, narrowed to clips where the transcript contains a specific phrase:

    ```python Python theme={null}
    index.documents.search(
        namespace="videos",
        top_k=10,
        score_by=[{"type": "dense_vector", "field": "frame_embedding", "values": query_embedding}],
        filter={"transcript": {"$match_phrase": "I love you"}},
    )
    ```

    `score_by` selects one ranking signal per request, but every signal stays addressable on the same documents.
  </Accordion>

  <Accordion title="Sparse + dense hybrid — single-vector index (vector API)">
    Use when you're modeling data with the [vector API](#records) (not the document API) and want to combine a sparse and dense vector in one record on a single index. For new document-centric projects with text data, prefer the document-shape Dense + FTS pattern above.

    ```json theme={null}
    {
      "id": "doc1#chunk1",
      "values": [0.0236, -0.0329, ..., -0.0104, 0.0086],
      "sparse_values": {
        "indices": [822745112, 1009084850, ...],
        "values":  [1.7958984, 0.41577148, ...]
      },
      "metadata": { "document_id": "doc1", "chunk_number": 1 }
    }
    ```

    See [Hybrid search](/guides/search/hybrid-search).
  </Accordion>
</AccordionGroup>

## Records

Records are how you model data for [indexes with dense vectors](/guides/get-started/concepts#index-with-dense-vectors) and [indexes with sparse vectors](/guides/get-started/concepts#index-with-sparse-vectors). Each record carries one vector (dense, sparse, or both for single-index hybrid) plus optional metadata, and you can upsert raw text in place of a vector when the index is [integrated with an embedding model](/guides/index-data/create-an-index#embedding-models).

<Tabs>
  <Tab title="Vectors">
    When you upsert pre-generated vectors, each record consists of the following:

    * **ID**: A unique string identifier for the record.
    * **Vector**: A dense vector for [semantic search](/guides/search/semantic-search), a sparse vector for [sparse-vector lexical search](/guides/search/lexical-search), or both for single-index [hybrid search](/guides/search/hybrid-search) (vector API).
    * **Metadata** (optional): A flat JSON document containing key-value pairs with additional information (nested objects are not supported). You can filter by metadata when searching or deleting records.

    <Note>
      When importing data from object storage, records must be in Parquet format. For more details, see [Import data](/guides/index-data/import-data#prepare-your-data).
    </Note>

    Example:

    <CodeGroup>
      ```json Dense theme={null}
      {
        "id": "document1#chunk1", 
        "values": [0.0236663818359375, -0.032989501953125, ..., -0.01041412353515625, 0.0086669921875], 
        "metadata": {
          "document_id": "document1",
          "document_title": "Introduction to Vector Databases",
          "chunk_number": 1,
          "chunk_text": "First chunk of the document content...",
          "document_url": "https://example.com/docs/document1",
          "created_at": "2024-01-15",
          "document_type": "tutorial"
        }
      }
      ```

      ```json Sparse theme={null}
      {
        "id": "document1#chunk1", 
        "sparse_values": {
          "values": [1.7958984, 0.41577148, ..., 4.4414062, 3.3554688],
          "indices": [822745112, 1009084850, ..., 3517203014, 3590924191]
        },
        "metadata": {
          "document_id": "document1",
          "document_title": "Introduction to Vector Databases",
          "chunk_number": 1,
          "chunk_text": "First chunk of the document content...",
          "document_url": "https://example.com/docs/document1",
          "created_at": "2024-01-15",
          "document_type": "tutorial"
        }
      }
      ```

      ```json Hybrid theme={null}
      {
        "id": "document1#chunk1", 
        "values": [0.0236663818359375, -0.032989501953125, ..., -0.01041412353515625, 0.0086669921875], 
        "sparse_values": {
          "values": [1.7958984, 0.41577148, ..., 4.4414062, 3.3554688],
          "indices": [822745112, 1009084850, ..., 3517203014, 3590924191]
        },
        "metadata": {
          "document_id": "document1",
          "document_title": "Introduction to Vector Databases",
          "chunk_number": 1,
          "chunk_text": "First chunk of the document content...",
          "document_url": "https://example.com/docs/document1",
          "created_at": "2024-01-15",
          "document_type": "tutorial"
        }
      }
      ```
    </CodeGroup>
  </Tab>

  <Tab title="Text">
    When you upsert raw text for Pinecone to convert to vectors automatically, each record consists of the following:

    * **ID**: A unique string identifier for the record.
    * **Text**: The raw text for Pinecone to convert to a dense vector for [semantic search](/guides/search/semantic-search) or a sparse vector for [sparse-vector lexical search](/guides/search/lexical-search), depending on the [embedding model](/guides/index-data/create-an-index#embedding-models) integrated with the index. This field name must match the `embed.field_map` defined in the index.
    * **Metadata** (optional): All additional fields are stored as record metadata. You can filter by metadata when searching or deleting records.

    <Note>
      Upserting raw text is supported only for [indexes with integrated embedding](/guides/index-data/indexing-overview#vector-embedding).
    </Note>

    Example:

    ```json theme={null}
    {
      "_id": "document1#chunk1", 
      "chunk_text": "First chunk of the document content...", // Text to convert to a vector. 
      "document_id": "document1", // This and subsequent fields stored as metadata. 
      "document_title": "Introduction to Vector Databases",
      "chunk_number": 1,
      "document_url": "https://example.com/docs/document1", 
      "created_at": "2024-01-15",
      "document_type": "tutorial"
    }
    ```
  </Tab>
</Tabs>

## Use structured IDs

Use a structured, human-readable format for record IDs, including ID prefixes that reflect the type of data you're storing, for example:

* **Document chunks**: `document_id#chunk_number`
* **User data**: `user_id#data_type#item_id`
* **Multi-tenant data**: `tenant_id#document_id#chunk_id`

Choose a delimiter for your ID prefixes that won't appear elsewhere in your IDs. Common patterns include:

* `document1#chunk1` - Using hash delimiter
* `document1_chunk1` - Using underscore delimiter
* `document1:chunk1` - Using colon delimiter

Structuring IDs in this way provides several advantages:

* **Efficiency**: Applications can quickly identify which record it should operate on.
* **Clarity**: Developers can easily understand what they're looking at when examining records.
* **Flexibility**: ID prefixes enable list operations for fetching and updating records.

## Include metadata

Include [metadata key-value pairs](/guides/index-data/indexing-overview#metadata) that support your application's key operations, for example:

* **Enable query-time filtering**: Add fields for time ranges, categories, or other criteria for [filtering searches for increased accuracy and relevance](/guides/search/filter-by-metadata).
* **Link related chunks**: Use fields like `document_id` and `chunk_number` to keep track of related records and enable efficient [chunk deletion](#delete-chunks) and [document updates](#update-an-entire-document).
* **Link back to original data**: Include `chunk_text` or `document_url` for traceability and user display.

Metadata keys must be strings, and metadata values must be one of the following data types:

* String
* Number (stored as a 64-bit floating point)
* Boolean (true, false)
* List of strings

<Note>
  Pinecone supports 40 KB of metadata per record.
</Note>

## Example

This example demonstrates how to manage document chunks in Pinecone using structured IDs and comprehensive metadata. It covers the complete lifecycle of chunked documents: upserting, searching, fetching, updating, and deleting chunks, and updating an entire document.

### Upsert chunks

When [upserting](/guides/index-data/upsert-data) documents that have been split into chunks, combine structured IDs with comprehensive metadata:

<Tabs>
  <Tab title="Upsert text">
    <Note>
      Upserting raw text is supported only for [indexes with integrated embedding](/guides/index-data/create-an-index#integrated-embedding).
    </Note>

    ```python Python theme={null}
    from pinecone.grpc import PineconeGRPC as Pinecone

    pc = Pinecone(api_key="YOUR_API_KEY")

    # To get the unique host for an index, 
    # see https://docs.pinecone.io/guides/manage-data/target-an-index
    index = pc.Index(host="INDEX_HOST")

    index.upsert_records(
      "example-namespace",
      [
        {
          "_id": "document1#chunk1", 
          "chunk_text": "First chunk of the document content...",
          "document_id": "document1",
          "document_title": "Introduction to Vector Databases",
          "chunk_number": 1,
          "document_url": "https://example.com/docs/document1",
          "created_at": "2024-01-15",
          "document_type": "tutorial"
        },
        {
          "_id": "document1#chunk2", 
          "chunk_text": "Second chunk of the document content...",
          "document_id": "document1",
          "document_title": "Introduction to Vector Databases", 
          "chunk_number": 2,
          "document_url": "https://example.com/docs/document1",
          "created_at": "2024-01-15",
          "document_type": "tutorial"
        },
        {
          "_id": "document1#chunk3", 
          "chunk_text": "Third chunk of the document content...",
          "document_id": "document1",
          "document_title": "Introduction to Vector Databases",
          "chunk_number": 3, 
          "document_url": "https://example.com/docs/document1",
          "created_at": "2024-01-15",
          "document_type": "tutorial"
        },
      ]
    )
    ```
  </Tab>

  <Tab title="Upsert vectors">
    ```python Python theme={null}
    from pinecone.grpc import PineconeGRPC as Pinecone

    pc = Pinecone(api_key="YOUR_API_KEY")

    # To get the unique host for an index, 
    # see https://docs.pinecone.io/guides/manage-data/target-an-index
    index = pc.Index(host="INDEX_HOST")

    index.upsert(
      namespace="example-namespace",
      vectors=[
        {
          "id": "document1#chunk1", 
          "values": [0.0236663818359375, -0.032989501953125, ..., -0.01041412353515625, 0.0086669921875], 
          "metadata": {
            "document_id": "document1",
            "document_title": "Introduction to Vector Databases",
            "chunk_number": 1,
            "chunk_text": "First chunk of the document content...",
            "document_url": "https://example.com/docs/document1",
            "created_at": "2024-01-15",
            "document_type": "tutorial"
          }
        },
        {
          "id": "document1#chunk2", 
          "values": [-0.0412445068359375, 0.028839111328125, ..., 0.01953125, -0.0174560546875],
          "metadata": {
            "document_id": "document1",
            "document_title": "Introduction to Vector Databases", 
            "chunk_number": 2,
            "chunk_text": "Second chunk of the document content...",
            "document_url": "https://example.com/docs/document1",
            "created_at": "2024-01-15",
            "document_type": "tutorial"
          }
        },
        {
          "id": "document1#chunk3", 
          "values": [0.0512237548828125, 0.041656494140625, ..., 0.02130126953125, -0.0394287109375],
          "metadata": {
            "document_id": "document1",
            "document_title": "Introduction to Vector Databases",
            "chunk_number": 3, 
            "chunk_text": "Third chunk of the document content...",
            "document_url": "https://example.com/docs/document1",
            "created_at": "2024-01-15",
            "document_type": "tutorial"
          }
        }
      ]
    )
    ```
  </Tab>
</Tabs>

### Search chunks

To search the chunks of a document, use a [metadata filter expression](/guides/search/filter-by-metadata#metadata-filter-expressions) that limits the search appropriately:

<Tabs>
  <Tab title="Search with text">
    <Note>
      Searching with text is supported only for [indexes with integrated embedding](/guides/index-data/create-an-index#integrated-embedding).
    </Note>

    ```python Python theme={null}
    from pinecone import Pinecone

    pc = Pinecone(api_key="YOUR_API_KEY")

    # To get the unique host for an index, 
    # see https://docs.pinecone.io/guides/manage-data/target-an-index
    index = pc.Index(host="INDEX_HOST")

    filtered_results = index.search(
        namespace="example-namespace", 
        query={
            "inputs": {"text": "What is a vector database?"}, 
            "top_k": 3,
            "filter": {"document_id": "document1"}
        },
        fields=["chunk_text"]
    )

    print(filtered_results)
    ```
  </Tab>

  <Tab title="Search with a vector">
    ```python Python theme={null}
    from pinecone.grpc import PineconeGRPC as Pinecone

    pc = Pinecone(api_key="YOUR_API_KEY")

    # To get the unique host for an index, 
    # see https://docs.pinecone.io/guides/manage-data/target-an-index
    index = pc.Index(host="INDEX_HOST")

    filtered_results = index.query(
        namespace="example-namespace",
        vector=[0.0236663818359375,-0.032989501953125, ..., -0.01041412353515625,0.0086669921875], 
        top_k=3,
        filter={
            "document_id": {"$eq": "document1"}
        },
        include_metadata=True,
        include_values=False
    )

    print(filtered_results)
    ```
  </Tab>
</Tabs>

### Fetch chunks

To retrieve all chunks for a specific document, first [list the record IDs](/guides/manage-data/list-record-ids) using the document prefix, and then [fetch](/guides/manage-data/fetch-data) the complete records:

```python Python theme={null}
from pinecone.grpc import PineconeGRPC as Pinecone

pc = Pinecone(api_key="YOUR_API_KEY")

# To get the unique host for an index, 
# see https://docs.pinecone.io/guides/manage-data/target-an-index
index = pc.Index(host="INDEX_HOST")

# List all chunks for document1 using ID prefix
chunk_ids = []
for record_id in index.list(prefix='document1#', namespace='example-namespace'):
    chunk_ids.append(record_id)

print(f"Found {len(chunk_ids)} chunks for document1")

# Fetch the complete records by ID
if chunk_ids:
    records = index.fetch(ids=chunk_ids, namespace='example-namespace')
    
    for record_id, record_data in records['vectors'].items():
        print(f"Chunk ID: {record_id}")
        print(f"Chunk text: {record_data['metadata']['chunk_text']}")
        # Process the vector values and metadata as needed
```

<Note>
  Pinecone is [eventually consistent](/guides/index-data/check-data-freshness), so it's possible that a write (upsert, update, or delete) followed immediately by a read (query, list, or fetch) may not return the latest version of the data. If your use case requires retrieving data immediately, consider implementing a small delay or [retry logic](/guides/production/error-handling#implement-retry-logic) after writes.
</Note>

### Update chunks

To [update](/guides/manage-data/update-data) specific chunks within a document, first list the chunk IDs, and then update individual records:

```python Python theme={null}
from pinecone.grpc import PineconeGRPC as Pinecone

pc = Pinecone(api_key="YOUR_API_KEY")

# To get the unique host for an index, 
# see https://docs.pinecone.io/guides/manage-data/target-an-index
index = pc.Index(host="INDEX_HOST")

# List all chunks for document1
chunk_ids = []
for record_id in index.list(prefix='document1#', namespace='example-namespace'):
    chunk_ids.append(record_id)

# Update specific chunks (e.g., update chunk 2)
if 'document1#chunk2' in chunk_ids:
    new_vector = ...  # from your embedding model
    index.update(
        id='document1#chunk2',
        values=new_vector,
        set_metadata={
            "document_id": "document1",
            "document_title": "Introduction to Vector Databases - Revised",
            "chunk_number": 2,
            "chunk_text": "Updated second chunk content...",
            "document_url": "https://example.com/docs/document1",
            "created_at": "2024-01-15",
            "updated_at": "2024-02-15",
            "document_type": "tutorial"
        },
        namespace='example-namespace'
    )
    print("Updated chunk 2 successfully")
```

### Delete chunks

To [delete](/guides/manage-data/delete-data#delete-records-by-metadata) chunks of a document, use a [metadata filter expression](/guides/search/filter-by-metadata#metadata-filter-expressions) that limits the deletion appropriately:

```python Python theme={null}
from pinecone.grpc import PineconeGRPC as Pinecone

pc = Pinecone(api_key="YOUR_API_KEY")

# To get the unique host for an index, 
# see https://docs.pinecone.io/guides/manage-data/target-an-index
index = pc.Index(host="INDEX_HOST")

# Delete chunks 1 and 3
index.delete(
    namespace="example-namespace",
    filter={
        "document_id": {"$eq": "document1"},
        "chunk_number": {"$in": [1, 3]}
    }
)

# Delete all chunks for a document
index.delete(
    namespace="example-namespace",
    filter={
        "document_id": {"$eq": "document1"}
    }
)
```

### Update an entire document

When the amount of chunks or ordering of chunks for a document changes, the recommended approach is to first [delete all chunks using a metadata filter](/guides/manage-data/delete-data#delete-records-by-metadata), and then [upsert](/guides/index-data/upsert-data) the new chunks:

```python Python theme={null}
from pinecone.grpc import PineconeGRPC as Pinecone

pc = Pinecone(api_key="YOUR_API_KEY")

# To get the unique host for an index, 
# see https://docs.pinecone.io/guides/manage-data/target-an-index
index = pc.Index(host="INDEX_HOST")

# Step 1: Delete all existing chunks for the document
index.delete(
    namespace="example-namespace",
    filter={
        "document_id": {"$eq": "document1"}
    }
)

print("Deleted existing chunks for document1")

# Step 2: Upsert the updated document chunks
chunk1_vector = ...  # from your embedding model
chunk2_vector = ...
index.upsert(
  namespace="example-namespace", 
  vectors=[
    {
      "id": "document1#chunk1",
      "values": chunk1_vector,
      "metadata": {
        "document_id": "document1",
        "document_title": "Introduction to Vector Databases - Updated Edition",
        "chunk_number": 1,
        "chunk_text": "Updated first chunk with new content...",
        "document_url": "https://example.com/docs/document1",
        "created_at": "2024-02-15",
        "document_type": "tutorial",
        "version": "2.0"
      }
    },
    {
      "id": "document1#chunk2",
      "values": chunk2_vector,
      "metadata": {
        "document_id": "document1",
        "document_title": "Introduction to Vector Databases - Updated Edition",
        "chunk_number": 2,
        "chunk_text": "Updated second chunk with new content...",
        "document_url": "https://example.com/docs/document1",
        "created_at": "2024-02-15",
        "document_type": "tutorial",
        "version": "2.0"
      }
    }
    # Add more chunks as needed for the updated document
  ]
)

print("Successfully updated document1 with new chunks")
```

## Data freshness

Pinecone is [eventually consistent](/guides/index-data/check-data-freshness), so it's possible that a write (upsert, update, or delete) followed immediately by a read (query, list, or fetch) may not return the latest version of the data. If your use case requires retrieving data immediately, consider implementing a small delay or [retry logic](/guides/production/error-handling#implement-retry-logic) after writes.

## Design for multi-tenancy

Many applications have a concept of tenants—users, organizations, projects, or other groups that should only access their own data. How you model this access control significantly impacts query performance and cost.

### Use namespaces for tenant isolation

The most efficient way to implement multi-tenancy is to use [namespaces](/guides/index-data/indexing-overview#namespaces) to separate data by tenant. With this approach, each tenant has their own namespace, and queries only scan that tenant's data—resulting in better performance and lower costs.

For a complete implementation guide with examples across all SDKs, see [Implement multitenancy](/guides/index-data/implement-multitenancy).

<Accordion title="Why namespaces are more efficient">
  When you use namespaces for multi-tenancy:

  * **Lower query costs and faster performance**: Query cost is based on namespace size. If you have 100 tenants with 1 GB each, querying one tenant's namespace costs 1 RU and scans only 1 GB. With metadata filtering in a single namespace (100 GB total), the same query costs 100 RUs and scans all 100 GB, even though the filter narrows results.
  * **Natural isolation**: Reduces the risk of application bugs that could query the wrong tenant's data (for example, by passing an incorrect filter value).
</Accordion>

### Avoid filtering by high-cardinality IDs

A common anti-pattern is storing all data in a single namespace and using metadata filters to scope queries to specific users:

<CodeGroup>
  ```python Python theme={null}
  # Anti-pattern: Filtering by many user IDs
  query_vector = [0.1, 0.2, 0.3, ...]  # Your query vector
  results = index.query(
      vector=query_vector,
      top_k=10,
      filter={
          "allowed_user_ids": {"$in": ["user_1", "user_2", ..., "user_10000"]}
      }
  )
  ```

  ```javascript JavaScript theme={null}
  // Anti-pattern: Filtering by many user IDs
  const queryVector = [0.1, 0.2, 0.3, ...];  // Your query vector
  const results = await index.query({
    vector: queryVector,
    topK: 10,
    filter: {
      allowed_user_ids: { $in: ["user_1", "user_2", ..., "user_10000"] }
    }
  });
  ```

  ```java Java theme={null}
  // Anti-pattern: Filtering by many user IDs
  import com.google.protobuf.Struct;
  import com.google.protobuf.Value;
  import io.pinecone.clients.Index;
  import io.pinecone.clients.Pinecone;
  import io.pinecone.unsigned_indices_model.QueryResponseWithUnsignedIndices;
  import java.util.Arrays;
  import java.util.List;

  Pinecone pinecone = new Pinecone.Builder("YOUR_API_KEY").build();
  Index index = pinecone.getIndexConnection("your-index-name");
  List<Float> queryVector = Arrays.asList(0.1f, 0.2f, 0.3f, ...);  // Your query vector

  // Build filter with $in operator (up to 10,000 values)
  Struct.Builder filterBuilder = Struct.newBuilder();
  Value.Builder listValueBuilder = Value.newBuilder();
  listValueBuilder.getListValueBuilder()
      .addAllValues(Arrays.asList(
          Value.newBuilder().setStringValue("user_1").build(),
          Value.newBuilder().setStringValue("user_2").build()
          // ... up to 10,000 values
      ));
  filterBuilder.putFields("allowed_user_ids",
      Value.newBuilder()
          .setStructValue(Struct.newBuilder()
              .putFields("$in", listValueBuilder.build())
              .build())
          .build());
  Struct filter = filterBuilder.build();

  QueryResponseWithUnsignedIndices results = index.queryByVector(
      10,
      queryVector,
      null, // default namespace
      filter
  );
  ```

  ```go Go theme={null}
  // Anti-pattern: Filtering by many user IDs
  import (
      "context"
      "fmt"
      "log"
      "github.com/pinecone-io/go-pinecone/v5/pinecone"
      "google.golang.org/protobuf/types/known/structpb"
  )

  ctx := context.Background()

  clientParams := pinecone.NewClientParams{
      ApiKey: "YOUR_API_KEY",
  }
  pc, err := pinecone.NewClient(clientParams)
  if err != nil {
      log.Fatalf("Failed to create Client: %v", err)
  }

  idx, err := pc.DescribeIndex(ctx, "your-index-name")
  if err != nil {
      log.Fatalf("Failed to describe index: %v", err)
  }

  idxConnection, err := pc.Index(pinecone.NewIndexConnParams{
      Host: idx.Host,
  })
  if err != nil {
      log.Fatalf("Failed to create IndexConnection: %v", err)
  }

  queryVector := []float32{0.1, 0.2, 0.3, ...}  // Your query vector

  userIds := []interface{}{"user_1", "user_2", /* ... up to 10,000 values */}
  metadataMap := map[string]interface{}{
      "allowed_user_ids": map[string]interface{}{
          "$in": userIds,
      },
  }
  filter, err := structpb.NewStruct(metadataMap)
  if err != nil {
      log.Fatalf("Failed to create filter: %v", err)
  }

  queryReq := &pinecone.QueryByVectorValuesRequest{
      Vector:          queryVector,
      TopK:            10,
      MetadataFilter:  filter,
      IncludeMetadata: true,
  }
  results, err := idxConnection.QueryByVectorValues(ctx, queryReq)
  if err != nil {
      log.Fatalf("Failed to query: %v", err)
  }
  fmt.Printf("Found %d matches:\n", len(results.Matches))
  for _, match := range results.Matches {
      fmt.Printf("  ID: %s, Score: %.4f\n", match.Vector.Id, match.Score)
      if match.Vector.Metadata != nil {
          fmt.Printf("    Metadata: %v\n", match.Vector.Metadata)
      }
  }
  ```

  ```csharp C# theme={null}
  // Anti-pattern: Filtering by many user IDs
  using Pinecone;
  using System;
  using System.Linq;

  var queryVector = new[] { 0.1f, 0.2f, 0.3f, ... };  // Your query vector
  // index is your IndexClient instance

  var userIds = new[] { "user_1", "user_2", /* ... up to 10,000 values */ };
  var filter = new Metadata
  {
      { "allowed_user_ids", new MetadataValue(new Metadata { { "$in", userIds } }) }
  };

  var results = await index.QueryAsync(new QueryRequest
  {
      Vector = queryVector,
      TopK = 10,
      Filter = filter,
      IncludeMetadata = true
  });
  if (results == null)
  {
      throw new InvalidOperationException("Failed to query");
  }
  Console.WriteLine($"Found {results.Matches?.Count() ?? 0} matches:");
  if (results.Matches != null)
  {
      foreach (var match in results.Matches)
      {
          Console.WriteLine($"  ID: {match.Id}, Score: {match.Score:F4}");
          if (match.Metadata != null)
          {
              Console.WriteLine($"    Metadata: {match.Metadata}");
          }
      }
  }
  ```

  ```bash curl theme={null}
  # Anti-pattern: Filtering by many user IDs
  PINECONE_API_KEY="YOUR_API_KEY"
  INDEX_HOST="INDEX_HOST"

  curl -X POST "https://$INDEX_HOST/query" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "Content-Type: application/json" \
       -H "X-Pinecone-Api-Version: 2025-10" \
       -d '{
             "vector": [0.1, 0.2, 0.3, ...],
             "topK": 10,
             "includeMetadata": true,
             "filter": {
               "allowed_user_ids": {
                 "$in": ["user_1", "user_2", ..., "user_10000"]
               }
             }
          }'
  ```
</CodeGroup>

This approach has several drawbacks:

* **Performance degradation**: Large `$in` filters increase network payload size and query latency.
* **Hard limits**: Each `$in` or `$nin` operator is limited to 10,000 values. Exceeding this limit will cause the request to fail. See [Metadata filter limits](/reference/api/database-limits#metadata-filter-limits).

### Use access control groups instead of individual IDs

If data must be shared across many tenants, design your access control using the smallest number of groups that describe a user's access:

<CodeGroup>
  ```python Python theme={null}
  # Better: Filter by organization or role instead of individual users
  query_vector = [0.1, 0.2, 0.3, ...]  # Your query vector
  results = index.query(
      vector=query_vector,
      top_k=10,
      filter={
          "$or": [
              {"organization_id": {"$eq": "org_A"}},
              {"project_id": {"$eq": "project_B"}}
          ]
      }
  )
  ```

  ```javascript JavaScript theme={null}
  // Better: Filter by organization or role instead of individual users
  const queryVector = [0.1, 0.2, 0.3, ...];  // Your query vector
  const results = await index.query({
    vector: queryVector,
    topK: 10,
    filter: {
      $or: [
        { organization_id: { $eq: "org_A" } },
        { project_id: { $eq: "project_B" } }
      ]
    }
  });
  ```

  ```java Java theme={null}
  // Better: Filter by organization or role instead of individual users
  import com.google.protobuf.Struct;
  import com.google.protobuf.Value;
  import io.pinecone.clients.Index;
  import io.pinecone.clients.Pinecone;
  import io.pinecone.unsigned_indices_model.QueryResponseWithUnsignedIndices;
  import java.util.Arrays;
  import java.util.List;

  Pinecone pinecone = new Pinecone.Builder("YOUR_API_KEY").build();
  Index index = pinecone.getIndexConnection("your-index-name");
  List<Float> queryVector = Arrays.asList(0.1f, 0.2f, 0.3f, ...);  // Your query vector

  // Build filter with $or operator
  Struct.Builder orgFilterBuilder = Struct.newBuilder();
  orgFilterBuilder.putFields("organization_id",
      Value.newBuilder()
          .setStructValue(Struct.newBuilder()
              .putFields("$eq", Value.newBuilder()
                  .setStringValue("org_A")
                  .build())
              .build())
          .build());

  Struct.Builder projectFilterBuilder = Struct.newBuilder();
  projectFilterBuilder.putFields("project_id",
      Value.newBuilder()
          .setStructValue(Struct.newBuilder()
              .putFields("$eq", Value.newBuilder()
                  .setStringValue("project_B")
                  .build())
              .build())
          .build());

  Struct.Builder orFilterBuilder = Struct.newBuilder();
  orFilterBuilder.putFields("$or",
      Value.newBuilder()
          .getListValueBuilder()
          .addValues(Value.newBuilder().setStructValue(orgFilterBuilder.build()).build())
          .addValues(Value.newBuilder().setStructValue(projectFilterBuilder.build()).build())
          .build());

  QueryResponseWithUnsignedIndices results = index.queryByVector(
      10,
      queryVector,
      null, // default namespace
      orFilterBuilder.build(),
      false, // includeValues
      true // includeMetadata
  );
  ```

  ```go Go theme={null}
  // Better: Filter by organization or role instead of individual users
  import (
      "context"
      "fmt"
      "log"
      "github.com/pinecone-io/go-pinecone/v5/pinecone"
      "google.golang.org/protobuf/types/known/structpb"
  )

  ctx := context.Background()

  clientParams := pinecone.NewClientParams{
      ApiKey: "YOUR_API_KEY",
  }
  pc, err := pinecone.NewClient(clientParams)
  if err != nil {
      log.Fatalf("Failed to create Client: %v", err)
  }

  idx, err := pc.DescribeIndex(ctx, "your-index-name")
  if err != nil {
      log.Fatalf("Failed to describe index: %v", err)
  }

  idxConnection, err := pc.Index(pinecone.NewIndexConnParams{
      Host: idx.Host,
  })
  if err != nil {
      log.Fatalf("Failed to create IndexConnection: %v", err)
  }

  queryVector := []float32{0.1, 0.2, 0.3, ...}  // Your query vector

  metadataMap := map[string]interface{}{
      "$or": []interface{}{
          map[string]interface{}{
              "organization_id": map[string]interface{}{
                  "$eq": "org_A",
              },
          },
          map[string]interface{}{
              "project_id": map[string]interface{}{
                  "$eq": "project_B",
              },
          },
      },
  }
  filter, err := structpb.NewStruct(metadataMap)
  if err != nil {
      log.Fatalf("Failed to create filter: %v", err)
  }

  queryReq := &pinecone.QueryByVectorValuesRequest{
      Vector:          queryVector,
      TopK:            10,
      MetadataFilter:  filter,
      IncludeMetadata: true,
  }
  results, err := idxConnection.QueryByVectorValues(ctx, queryReq)
  if err != nil {
      log.Fatalf("Failed to query: %v", err)
  }
  fmt.Printf("Found %d matches:\n", len(results.Matches))
  for _, match := range results.Matches {
      fmt.Printf("  ID: %s, Score: %.4f\n", match.Vector.Id, match.Score)
      if match.Vector.Metadata != nil {
          fmt.Printf("    Metadata: %v\n", match.Vector.Metadata)
      }
  }
  ```

  ```csharp C# theme={null}
  // Better: Filter by organization or role instead of individual users
  using Pinecone;
  using System;
  using System.Linq;

  var queryVector = new[] { 0.1f, 0.2f, 0.3f, ... };  // Your query vector
  // index is your IndexClient instance

  var filter = new Metadata
  {
      {
          "$or",
          new MetadataValue(new[]
          {
              new MetadataValue(new Metadata
              {
                  {
                      "organization_id",
                      new MetadataValue(new Metadata { { "$eq", "org_A" } })
                  }
              }),
              new MetadataValue(new Metadata
              {
                  {
                      "project_id",
                      new MetadataValue(new Metadata { { "$eq", "project_B" } })
                  }
              })
          })
      }
  };

  var results = await index.QueryAsync(new QueryRequest
  {
      Vector = queryVector,
      TopK = 10,
      Filter = filter,
      IncludeMetadata = true
  });
  if (results == null)
  {
      throw new InvalidOperationException("Failed to query");
  }
  Console.WriteLine($"Found {results.Matches?.Count() ?? 0} matches:");
  if (results.Matches != null)
  {
      foreach (var match in results.Matches)
      {
          Console.WriteLine($"  ID: {match.Id}, Score: {match.Score:F4}");
          if (match.Metadata != null)
          {
              Console.WriteLine($"    Metadata: {match.Metadata}");
          }
      }
  }
  ```

  ```bash curl theme={null}
  # Better: Filter by organization or role instead of individual users
  PINECONE_API_KEY="YOUR_API_KEY"
  INDEX_HOST="INDEX_HOST"

  curl -X POST "https://$INDEX_HOST/query" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "Content-Type: application/json" \
       -H "X-Pinecone-Api-Version: 2025-10" \
       -d '{
             "vector": [0.1, 0.2, 0.3, ...],
             "topK": 10,
             "includeMetadata": true,
             "filter": {
               "$or": [
                 {"organization_id": {"$eq": "org_A"}},
                 {"project_id": {"$eq": "project_B"}}
               ]
             }
          }'
  ```
</CodeGroup>

Instead of passing thousands of user IDs, this filter uses only 2 group identifiers to achieve the same access control.

### Multitenancy patterns

The following table provides general guidelines for choosing a multitenancy approach. Evaluate your specific use case, access patterns, and requirements to determine the best fit for your application.

| Data pattern                              | Recommended approach                                                  | Query cost                             | Performance |
| :---------------------------------------- | :-------------------------------------------------------------------- | :------------------------------------- | :---------- |
| Each tenant's data is completely separate | One index, one namespace per tenant                                   | Lowest (scans only tenant namespace)   | Fastest     |
| Large tenants with many sub-groups        | One index per large tenant, namespaces for sub-groups                 | Low (scans only sub-group namespace)   | Fast        |
| Data shared across tenants                | One index, shared namespace, filter by group IDs (org, project, role) | Higher (scans entire shared namespace) | Slower      |

<Warning>
  Avoid filtering by large lists of individual user IDs (for example, `{"user_id": {"$in": ["user_1", "user_2", ..., "user_10000"]}}`). This approach has the following drawbacks:

  * Hard limits: Each `$in` or `$nin` operator is limited to 10,000 values. Exceeding this limit will cause requests to fail.
  * Performance: Large filters increase query latency.
  * Higher costs: You pay for scanning the entire shared namespace, even though the filter narrows results.

  Instead, consider these alternatives:

  * Use one namespace per tenant (see row 1 in the table above).
  * Filter by broader groups like organization, project, or role rather than individual user IDs (see row 3 in the table above).
  * Retrieve a larger top K without filtering (for example, top 1000), then filter the results client-side.
</Warning>

For a complete step-by-step implementation guide, see [Implement multitenancy](/guides/index-data/implement-multitenancy).
