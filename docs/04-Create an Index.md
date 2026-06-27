> ## Documentation Index
> Fetch the complete documentation index at: https://docs.pinecone.io/llms.txt
> Use this file to discover all available pages before exploring further.
>
> Hello
> Hello

# Create an index File

> Create indexes for full-text, semantic, lexical, and hybrid search.

<script type="application/ld+json">
  {`
    [
    {
      "@context": "https://schema.org",
      "@type": "WebSite",
      "name": "Pinecone Documentation",
      "url": "https://docs.pinecone.io",
      "description": "Official documentation for Pinecone, the vector database for AI applications.",
      "publisher": {
        "@type": "Organization",
        "name": "Pinecone",
        "url": "https://www.pinecone.io"
      },
      "potentialAction": {
        "@type": "SearchAction",
        "target": {
          "@type": "EntryPoint",
          "urlTemplate": "https://docs.pinecone.io/search?query={search_term_string}"
        },
        "query-input": "required name=search_term_string"
      }
    },
    {
      "@context": "https://schema.org",
      "@type": "Organization",
      "name": "Pinecone",
      "url": "https://www.pinecone.io",
      "logo": "https://www.pinecone.io/images/docs_og_image_v2.png",
      "description": "Pinecone is the vector database built for AI. Search through billions of items for similar matches to any object, in milliseconds.",
      "sameAs": [
        "https://x.com/pinecone",
        "https://www.linkedin.com/company/pinecone-io",
        "https://github.com/pinecone-io"
      ],
      "knowsAbout": [
        "Vector databases",
        "Semantic search",
        "AI infrastructure",
        "Machine learning"
      ]
    }
    ]
    `}
</script>

A Pinecone index can hold any combination of the following:

* **Documents** are the unit of data in an index with a document schema — JSON records whose ranking fields are indexed according to a schema you declare at index creation. An index with a document schema can mix `dense_vector`, `sparse_vector`, and FTS-enabled `string` ranking fields in the same record, alongside any number of metadata fields (auto-indexed at upsert time). Use documents for [full-text search](/guides/search/full-text-search) (BM25 ranking on `string` fields with `full_text_search` enabled), and to combine multiple scoring methods on the same data via `score_by`.

* **Dense vectors** are numerical representations of the meaning and relationships of text, images, or other data. Indexes of dense vectors are used for [semantic search](/guides/search/semantic-search), or together with sparse vectors for [hybrid search](/guides/search/hybrid-search).

* **Sparse vectors** are high-dimensional vectors with mostly zero values, produced by a sparse embedding model such as [`pinecone-sparse-english-v0`](/models/pinecone-sparse-english-v0). Indexes of sparse vectors are used for [sparse-vector lexical search](/guides/search/lexical-search), or together with dense vectors for [hybrid search](/guides/search/hybrid-search).

<Tip>
  You can create an index using the [Pinecone console](https://app.pinecone.io/organizations/-/projects/-/create-index/serverless).
</Tip>

## Create an index for full-text search

An index with a document schema stores typed JSON documents. The schema declares how each ranking field is indexed: as a `string` field with `full_text_search` enabled for BM25 ranking, a `dense_vector` for ANN similarity, or a `sparse_vector`. A single index can mix all three ranking field types; at query time, pick the ranking signal with `score_by`. Metadata fields (anything else you upsert) are not declared in the schema — they're auto-indexed for filtering at upsert time.

<Note>
  Full-text search is not integrated embedding. A `string` field with `full_text_search` is indexed for BM25 ranking and Lucene queries. It does not call an embedding model. Integrated embedding remains available for vector API indexes.
</Note>

Indexes with document schemas are in [public preview](/guides/search/full-text-search#public-preview) and use API version `2026-01.alpha`. The preview supports REST and the Python SDK; for other languages, call the REST endpoint directly.

### Minimal: BM25 on a single text field

The example below creates an `articles` index whose `body` field is indexed for BM25 ranking. Other fields included at upsert time are stored on each document and auto-indexed for filtering as metadata.

```bash curl theme={null}
curl -X POST "https://api.pinecone.io/indexes" \
  -H "Api-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Pinecone-Api-Version: 2026-01.alpha" \
  -d '{
    "name": "articles",
    "deployment": {
      "deployment_type": "managed",
      "cloud": "aws",
      "region": "us-east-1"
    },
    "schema": {
      "fields": {
        "body": {
          "type": "string",
          "full_text_search": {}
        }
      }
    }
  }'
```

### Multi-field schema: BM25 + dense vector

A single index with a document schema can hold FTS-enabled `string` and `dense_vector` ranking fields together (the same schema can also include a `sparse_vector` field). A single search request ranks by one scoring type — multi-field BM25 is supported (multiple `text` clauses on different fields, or one `query_string` clause spanning fields), and any scoring method can be combined with metadata filters, including text-match filters (`$match_phrase`, `$match_all`, `$match_any`) on FTS-enabled `string` fields.

```bash curl theme={null}
curl -X POST "https://api.pinecone.io/indexes" \
  -H "Api-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "X-Pinecone-Api-Version: 2026-01.alpha" \
  -d '{
    "name": "articles-multi",
    "deployment": {
      "deployment_type": "managed",
      "cloud": "aws",
      "region": "us-east-1"
    },
    "schema": {
      "fields": {
        "title":    { "type": "string", "full_text_search": {} },
        "body":     { "type": "string", "full_text_search": {} },
        "embedding":{ "type": "dense_vector", "dimension": 1536, "metric": "cosine" }
      }
    }
  }'
```

You can include additional fields (for example, `category` or `year`) at upsert time. All metadata fields are automatically indexed for filtering — they don't need to be declared in the schema. The schema is for ranking fields only; declaring a metadata-only field (`string` without `full_text_search`, `string_list`, `float`, or `boolean`) is rejected at index creation.

For the full schema reference (all field types, language and analyzer options, dedicated read capacity, and Python SDK examples), see [Full-text search](/guides/search/full-text-search).

<Warning>
  Schema migration is not yet supported. Once an index with a document schema is created, you cannot add, remove, or modify fields. Plan your schema carefully — if you need to change a schema, [delete the index](/guides/manage-data/manage-indexes#delete-an-index) and create a new one.
</Warning>

## Create an index for dense vectors

You can create an index that stores dense vectors with [integrated vector embedding](/guides/index-data/indexing-overview#integrated-embedding), or one that stores vectors generated with an external embedding model.

### Integrated embedding

<Note>
  Indexes with integrated embedding do not support [updating](/guides/manage-data/update-data) or [importing](/guides/index-data/import-data) with text.
</Note>

If you want to upsert and search with source text and have Pinecone convert it to dense vectors automatically, [create an index with integrated embedding](/reference/api/latest/control-plane/create_for_model) as follows:

* Provide a `name` for the index.
* Set `cloud` and `region` to the [cloud and region](/guides/index-data/create-an-index#cloud-regions) where the index should be deployed.
* Set `embed.model` to one of [Pinecone's hosted embedding models](/guides/index-data/create-an-index#embedding-models).
* Set `embed.field_map` to the name of the field in your source document that contains the data for embedding.

Other parameters are optional. See the [API reference](/reference/api/latest/control-plane/create_for_model) for details.

<CodeGroup>
  ```python Python theme={null}
  from pinecone import Pinecone

  pc = Pinecone(api_key="YOUR_API_KEY")

  index_name = "integrated-dense-py"

  if not pc.has_index(index_name):
      pc.create_index_for_model(
          name=index_name,
          cloud="aws",
          region="us-east-1",
          embed={
              "model":"llama-text-embed-v2",
              "field_map":{"text": "chunk_text"}
          }
      )
  ```

  ```javascript JavaScript theme={null}
  import { Pinecone } from '@pinecone-database/pinecone'

  const pc = new Pinecone({ apiKey: 'YOUR_API_KEY' });

  await pc.createIndexForModel({
    name: 'integrated-dense-js',
    cloud: 'aws',
    region: 'us-east-1',
    embed: {
      model: 'llama-text-embed-v2',
      fieldMap: { text: 'chunk_text' },
    },
    waitUntilReady: true,
  });
  ```

  ```java Java theme={null}
  import io.pinecone.clients.Pinecone;
  import org.openapitools.db_control.client.ApiException;
  import org.openapitools.db_control.client.model.CreateIndexForModelRequest;
  import org.openapitools.db_control.client.model.CreateIndexForModelRequestEmbed;
  import org.openapitools.db_control.client.model.DeletionProtection;
  import java.util.HashMap;
  import java.util.Map;

  public class CreateIntegratedIndex {
      public static void main(String[] args) throws ApiException {
          Pinecone pc = new Pinecone.Builder("YOUR_API_KEY").build();
          String indexName = "integrated-dense-java";
          String region = "us-east-1";
          HashMap<String, String> fieldMap = new HashMap<>();
          fieldMap.put("text", "chunk_text");
          CreateIndexForModelRequestEmbed embed = new CreateIndexForModelRequestEmbed()
                  .model("llama-text-embed-v2")
                  .fieldMap(fieldMap);
          Map<String, String> tags = new HashMap<>();
          tags.put("environment", "development");
          pc.createIndexForModel(
                  indexName,
                  CreateIndexForModelRequest.CloudEnum.AWS,
                  region,
                  embed,
                  DeletionProtection.DISABLED,
                  tags
          );
      }
  }
  ```

  ```go Go theme={null}
  package main

  import (
      "context"
      "fmt"
      "log"

      "github.com/pinecone-io/go-pinecone/v4/pinecone"
  )

  func main() {
      ctx := context.Background()

      pc, err := pinecone.NewClient(pinecone.NewClientParams{
          ApiKey: "YOUR_API_KEY",
      })
      if err != nil {
          log.Fatalf("Failed to create Client: %v", err)
      }

    	indexName := "integrated-dense-go"
     	deletionProtection := pinecone.DeletionProtectionDisabled

      idx, err := pc.CreateIndexForModel(ctx, &pinecone.CreateIndexForModelRequest{
  		Name:   indexName,
  		Cloud:  pinecone.Aws,
  		Region: "us-east-1",
  		Embed: pinecone.CreateIndexForModelEmbed{
  			Model:    "llama-text-embed-v2",
  			FieldMap: map[string]interface{}{"text": "chunk_text"},
  		},
          DeletionProtection: &deletionProtection,
          Tags:   &pinecone.IndexTags{ "environment": "development" },
  	})
      if err != nil {
          log.Fatalf("Failed to create serverless integrated index: %v", idx.Name)
      } else {
          fmt.Printf("Successfully created serverless integrated index: %v", idx.Name)
      }
  }
  ```

  ```csharp C# theme={null}
  using Pinecone;

  var pinecone = new PineconeClient("YOUR_API_KEY");

  var createIndexRequest = await pinecone.CreateIndexForModelAsync(
      new CreateIndexForModelRequest
      {
          Name = "integrated-dense-dotnet",
          Cloud = CreateIndexForModelRequestCloud.Aws,
          Region = "us-east-1",
          Embed = new CreateIndexForModelRequestEmbed
          {
              Model = "llama-text-embed-v2",
              FieldMap = new Dictionary<string, object?>() 
              { 
                  { "text", "chunk_text" } 
              }
          },
          DeletionProtection = DeletionProtection.Disabled,
          Tags = new Dictionary<string, string> 
          { 
              { "environment", "development" }
          }
      }
  );
  ```

  ```json curl theme={null}
  PINECONE_API_KEY="YOUR_API_KEY"

  curl -X POST "https://api.pinecone.io/indexes/create-for-model" \
       -H "Content-Type: application/json" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "X-Pinecone-Api-Version: 2025-10" \
       -d '{
             "name": "integrated-dense-curl",
             "cloud": "aws",
             "region": "us-east-1",
             "embed": {
               "model": "llama-text-embed-v2",
               "field_map": {
                 "text": "chunk_text"
               }
             }
           }'
  ```

  ```bash CLI theme={null}
  # Target the project where you want to create the index.
  pc target -o "example-org" -p "example-project"
  # Create the index.
  pc index create \
    --name "integrated-dense-cli" \
    --metric "cosine" \
    --cloud "aws" \
    --region "us-east-1" \
    --model "llama-text-embed-v2" \
    --field_map "text=chunk_text" \
    --tags "environment=development"
  ```
</CodeGroup>

### Bring your own vectors

If you use an external embedding model to convert your data to dense vectors, [create an index](/reference/api/latest/control-plane/create_index) as follows:

* Provide a `name` for the index.
* Set the `vector_type` to `dense`.
* Specify the `dimension` and similarity `metric` of the vectors you'll store in the index. This should match the dimension and metric supported by your embedding model.
* Set `spec.cloud` and `spec.region` to the [cloud and region](/guides/index-data/create-an-index#cloud-regions) where the index should be deployed. For Python, you also need to import the `ServerlessSpec` class.

Other parameters are optional. See the [API reference](/reference/api/latest/control-plane/create_index) for details.

<CodeGroup>
  ```python Python theme={null}
  from pinecone.grpc import PineconeGRPC as Pinecone
  from pinecone import ServerlessSpec

  pc = Pinecone(api_key="YOUR_API_KEY")

  index_name = "standard-dense-py"

  if not pc.has_index(index_name):
      pc.create_index(
          name=index_name,
          vector_type="dense",
          dimension=1536,
          metric="cosine",
          spec=ServerlessSpec(
              cloud="aws",
              region="us-east-1"
          ),
          deletion_protection="disabled",
          tags={
              "environment": "development"
          }
      )
  ```

  ```javascript JavaScript theme={null}
  import { Pinecone } from '@pinecone-database/pinecone'

  const pc = new Pinecone({ apiKey: 'YOUR_API_KEY' });

  await pc.createIndex({
    name: 'standard-dense-js',
    vectorType: 'dense',
    dimension: 1536,
    metric: 'cosine',
    spec: {
      serverless: {
        cloud: 'aws',
        region: 'us-east-1'
      }
    },
    deletionProtection: 'disabled',
    tags: { environment: 'development' }, 
  });
  ```

  ```java Java theme={null}
  import io.pinecone.clients.Pinecone;
  import org.openapitools.db_control.client.model.IndexModel;
  import org.openapitools.db_control.client.model.DeletionProtection;
  import java.util.HashMap;

  public class CreateServerlessIndexExample {
      public static void main(String[] args) {
          Pinecone pc = new Pinecone.Builder("YOUR_API_KEY").build();
          String indexName = "standard-dense-java";
          String cloud = "aws";
          String region = "us-east-1";
          String vectorType = "dense";
          Map<String, String> tags = new HashMap<>();
          tags.put("environment", "development");
          pc.createServerlessIndex(
              indexName,
              "cosine", 
              1536, 
              cloud,
              region,
              DeletionProtection.DISABLED, 
              tags, 
              vectorType
          );
      }
  }
  ```

  ```go Go theme={null}
  package main

  import (
      "context"
      "fmt"
      "log"

      "github.com/pinecone-io/go-pinecone/v4/pinecone"
  )

  func main() {
      ctx := context.Background()

      pc, err := pinecone.NewClient(pinecone.NewClientParams{
          ApiKey: "YOUR_API_KEY",
      })
      if err != nil {
          log.Fatalf("Failed to create Client: %v", err)
      }

      // Serverless index
    	indexName := "standard-dense-go"
  	vectorType := "dense"
      dimension := int32(1536)
      metric := pinecone.Cosine
  	deletionProtection := pinecone.DeletionProtectionDisabled

      idx, err := pc.CreateServerlessIndex(ctx, &pinecone.CreateServerlessIndexRequest{
          Name:               indexName,
          VectorType:         &vectorType,
          Dimension:          &dimension,
          Metric:             &metric,
          Cloud:              pinecone.Aws,
          Region:             "us-east-1",
          DeletionProtection: &deletionProtection,
          Tags:               &pinecone.IndexTags{ "environment": "development" },
      })
      if err != nil {
          log.Fatalf("Failed to create serverless index: %v", err)
      } else {
          fmt.Printf("Successfully created serverless index: %v", idx.Name)
      }
  }
  ```

  ```csharp C# theme={null}
  using Pinecone;

  var pinecone = new PineconeClient("YOUR_API_KEY");

  var createIndexRequest = await pinecone.CreateIndexAsync(new CreateIndexRequest
  {
      Name = "standard-dense-dotnet",
      VectorType = VectorType.Dense,
      Dimension = 1536,
      Metric = MetricType.Cosine,
      Spec = new ServerlessIndexSpec
      {
          Serverless = new ServerlessSpec
          {
              Cloud = ServerlessSpecCloud.Aws,
              Region = "us-east-1"
          }
      },
      DeletionProtection = DeletionProtection.Disabled,
      Tags = new Dictionary<string, string> 
      {  
          { "environment", "development" }
      }
  });
  ```

  ```shell curl theme={null}
  PINECONE_API_KEY="YOUR_API_KEY"

  curl -X POST "https://api.pinecone.io/indexes" \
       -H "Accept: application/json" \
       -H "Content-Type: application/json" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "X-Pinecone-Api-Version: 2025-10" \
       -d '{
             "name": "standard-dense-curl",
             "vector_type": "dense",
             "dimension": 1536,
             "metric": "cosine",
             "spec": {
               "serverless": {
                 "cloud": "aws",
                 "region": "us-east-1"
                }
              },
              "tags": {
                "environment": "development"
              },
              "deletion_protection": "disabled"
           }'
  ```

  ```bash CLI theme={null}
  # Target the project where you want to create the index.
  pc target -o "example-org" -p "example-project"
  # Create the index.
  pc index create \
    --name "standard-dense-cli" \
    --vector_type "dense" \
    --dimension 1536 \
    --metric "cosine" \
    --cloud "aws" \
    --region "us-east-1" \
    --tags "environment=development" \
    --deletion_protection "disabled"
  ```
</CodeGroup>

## Create an index for sparse vectors

You can create an index that stores sparse vectors with [integrated vector embedding](/guides/index-data/indexing-overview#integrated-embedding), or one that stores vectors generated with an external embedding model.

### Integrated embedding

If you want to upsert and search with source text and have Pinecone convert it to sparse vectors automatically, [create an index with integrated embedding](/reference/api/latest/control-plane/create_for_model) as follows:

* Provide a `name` for the index.
* Set `cloud` and `region` to the [cloud and region](/guides/index-data/create-an-index#cloud-regions) where the index should be deployed.
* Set `embed.model` to one of [Pinecone's hosted sparse embedding models](/guides/index-data/create-an-index#embedding-models).
* Set `embed.field_map` to the name of the field in your source document that contains the text for embedding.
* If needed, `embed.read_parameters` and `embed.write_parameters` can be used to override the default model embedding behavior.

Other parameters are optional. See the [API reference](/reference/api/latest/control-plane/create_for_model) for details.

<CodeGroup>
  ```python Python theme={null}
  from pinecone import Pinecone

  pc = Pinecone(api_key="YOUR_API_KEY")

  index_name = "integrated-sparse-py"

  if not pc.has_index(index_name):
      pc.create_index_for_model(
          name=index_name,
          cloud="aws",
          region="us-east-1",
          embed={
              "model":"pinecone-sparse-english-v0",
              "field_map":{"text": "chunk_text"}
          }
      )
  ```

  ```javascript JavaScript theme={null}
  import { Pinecone } from '@pinecone-database/pinecone'

  const pc = new Pinecone({ apiKey: 'YOUR_API_KEY' });

  await pc.createIndexForModel({
    name: 'integrated-sparse-js',
    cloud: 'aws',
    region: 'us-east-1',
    embed: {
      model: 'pinecone-sparse-english-v0',
      fieldMap: { text: 'chunk_text' },
    },
    waitUntilReady: true,
  });
  ```

  ```java Java theme={null}
  import io.pinecone.clients.Pinecone;
  import org.openapitools.db_control.client.ApiException;
  import org.openapitools.db_control.client.model.CreateIndexForModelRequest;
  import org.openapitools.db_control.client.model.CreateIndexForModelRequestEmbed;
  import org.openapitools.db_control.client.model.DeletionProtection;
  import java.util.HashMap;
  import java.util.Map;

  public class CreateIntegratedIndex {
      public static void main(String[] args) throws ApiException {
          Pinecone pc = new Pinecone.Builder("YOUR_API_KEY").build();
          String indexName = "integrated-sparse-java";
          String region = "us-east-1";
          HashMap<String, String> fieldMap = new HashMap<>();
          fieldMap.put("text", "chunk_text");
          CreateIndexForModelRequestEmbed embed = new CreateIndexForModelRequestEmbed()
                  .model("pinecone-sparse-english-v0")
                  .fieldMap(fieldMap);
          Map<String, String> tags = new HashMap<>();
          tags.put("environment", "development");
          pc.createIndexForModel(
                  indexName,
                  CreateIndexForModelRequest.CloudEnum.AWS,
                  region,
                  embed,
                  DeletionProtection.DISABLED,
                  tags
          );
      }
  }
  ```

  ```go Go theme={null}
  package main

  import (
      "context"
      "fmt"
      "log"

      "github.com/pinecone-io/go-pinecone/v4/pinecone"
  )

  func main() {
      ctx := context.Background()

      pc, err := pinecone.NewClient(pinecone.NewClientParams{
          ApiKey: "YOUR_API_KEY",
      })
      if err != nil {
          log.Fatalf("Failed to create Client: %v", err)
      }

    	indexName := "integrated-sparse-go"
  	deletionProtection := pinecone.DeletionProtectionDisabled

      idx, err := pc.CreateIndexForModel(ctx, &pinecone.CreateIndexForModelRequest{
  		Name:   indexName,
  		Cloud:  pinecone.Aws,
  		Region: "us-east-1",
  		Embed: pinecone.CreateIndexForModelEmbed{
  			Model:    "pinecone-sparse-english-v0",
  			FieldMap: map[string]interface{}{"text": "chunk_text"},
  		},
          DeletionProtection: &deletionProtection,
          Tags:   &pinecone.IndexTags{ "environment": "development" },

  	})
      if err != nil {
          log.Fatalf("Failed to create serverless integrated index: %v", idx.Name)
      } else {
          fmt.Printf("Successfully created serverless integrated index: %v", idx.Name)
      }
  }
  ```

  ```csharp C# theme={null}
  using Pinecone;

  var pinecone = new PineconeClient("YOUR_API_KEY");

  var createIndexRequest = await pinecone.CreateIndexForModelAsync(
      new CreateIndexForModelRequest
      {
          Name = "integrated-sparse-dotnet",
          Cloud = CreateIndexForModelRequestCloud.Aws,
          Region = "us-east-1",
          Embed = new CreateIndexForModelRequestEmbed
          {
              Model = "pinecone-sparse-english-v0",
              FieldMap = new Dictionary<string, object?>() 
              { 
                  { "text", "chunk_text" } 
              }
          },
          DeletionProtection = DeletionProtection.Disabled,
          Tags = new Dictionary<string, string> 
          { 
              { "environment", "development" }
          }
      }
  );
  ```

  ```shell curl theme={null}
  PINECONE_API_KEY="YOUR_API_KEY"

  curl -X POST "https://api.pinecone.io/indexes/create-for-model" \
       -H "Content-Type: application/json" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "X-Pinecone-Api-Version: 2025-10" \
       -d '{
             "name": "integrated-sparse-curl",
             "cloud": "aws",
             "region": "us-east-1",
             "embed": {
               "model": "pinecone-sparse-english-v0",
               "field_map": {
                 "text": "chunk_text"
               }
             }
           }'
  ```

  ```bash CLI theme={null}
  # Target the project where you want to create the index.
  pc target -o "example-org" -p "example-project"
  # Create the index.
  pc index create \
    --name "integrated-sparse-cli" \
    --cloud "aws" \
    --region "us-east-1" \
    --model "pinecone-sparse-english-v0" \
    --field_map "text=chunk_text" \
    --tags "environment=development"
  ```
</CodeGroup>

### Bring your own vectors

If you use an external embedding model to convert your data to sparse vectors, [create an index](/reference/api/latest/control-plane/create_index) as follows:

* Provide a `name` for the index.
* Set the `vector_type` to `sparse`.
* Set the distance `metric` to `dotproduct`. Indexes that store sparse vectors do not support other [distance metrics](/guides/index-data/indexing-overview#distance-metrics).
* Set `spec.cloud` and `spec.region` to the cloud and region where the index should be deployed.

Other parameters are optional. See the [API reference](/reference/api/latest/control-plane/create_index) for details.

<CodeGroup>
  ```python Python theme={null}
  from pinecone import Pinecone, ServerlessSpec

  pc = Pinecone(api_key="YOUR_API_KEY")

  index_name = "standard-sparse-py"

  if not pc.has_index(index_name):
      pc.create_index(
          name=index_name,
          vector_type="sparse",
          metric="dotproduct",
          spec=ServerlessSpec(cloud="aws", region="us-east-1")
      )
  ```

  ```javascript JavaScript theme={null}
  import { Pinecone } from '@pinecone-database/pinecone'

  const pc = new Pinecone({ apiKey: 'YOUR_API_KEY' });

  await pc.createIndex({
    name: 'standard-sparse-js',
    vectorType: 'sparse',
    metric: 'dotproduct',
    spec: {
      serverless: {
        cloud: 'aws',
        region: 'us-east-1'
      },
    },
  });
  ```

  ```java Java theme={null}
  import io.pinecone.clients.Pinecone;
  import org.openapitools.db_control.client.model.DeletionProtection;

  import java.util.*;

  public class SparseIndex {
      public static void main(String[] args) throws InterruptedException {
          // Instantiate Pinecone class
          Pinecone pinecone = new Pinecone.Builder("YOUR_API_KEY").build();

          // Create the index
          String indexName = "standard-sparse-java";
          String cloud = "aws";
          String region = "us-east-1";
          String vectorType = "sparse";
          Map<String, String> tags = new HashMap<>();
          tags.put("env", "test");
          pinecone.createSparseServelessIndex(indexName,
                  cloud,
                  region,
                  DeletionProtection.DISABLED,
                  tags,
                  vectorType);
      }
  }
  ```

  ```go Go theme={null}
  package main

  import (
  	"context"
  	"fmt"
  	"log"

  	"github.com/pinecone-io/go-pinecone/v4/pinecone"
  )

  func main() {
  	ctx := context.Background()

  	pc, err := pinecone.NewClient(pinecone.NewClientParams{
  		ApiKey: "YOUR_API_KEY",
  	})
  	if err != nil {
  		log.Fatalf("Failed to create Client: %v", err)
  	}

  	indexName := "standard-sparse-go"
  	vectorType := "sparse"
  	metric := pinecone.Dotproduct
  	deletionProtection := pinecone.DeletionProtectionDisabled

  	idx, err := pc.CreateServerlessIndex(ctx, &pinecone.CreateServerlessIndexRequest{
  		Name:               indexName,
  		Metric:             &metric,
  		VectorType:         &vectorType,
  		Cloud:              pinecone.Aws,
  		Region:             "us-east-1",
  		DeletionProtection: &deletionProtection,
  	})
  	if err != nil {
  		log.Fatalf("Failed to create serverless index: %v", err)
  	} else {
  		fmt.Printf("Successfully created serverless index: %v", idx.Name)
  	}
  }
  ```

  ```csharp C# theme={null}
  using Pinecone;

  var pinecone = new PineconeClient("YOUR_API_KEY");

  var createIndexRequest = await pinecone.CreateIndexAsync(new CreateIndexRequest
  {
      Name = "standard-sparse-dotnet",
      VectorType = VectorType.Sparse,
      Metric = MetricType.Dotproduct,
      Spec = new ServerlessIndexSpec
      {
          Serverless = new ServerlessSpec
          {
              Cloud = ServerlessSpecCloud.Aws,
              Region = "us-east-1"
          }
      }
  });
  ```

  ```shell curl theme={null}
  PINECONE_API_KEY="YOUR_API_KEY"

  curl -X POST "https://api.pinecone.io/indexes" \
       -H "Accept: application/json" \
       -H "Content-Type: application/json" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "X-Pinecone-Api-Version: 2025-10" \
       -d '{
              "name": "standard-sparse-curl",
              "vector_type": "sparse",
              "metric": "dotproduct",
              "spec": {
                 "serverless": {
                    "cloud": "aws",
                    "region": "us-east-1"
                 }
              }
           }'
  ```

  ```bash CLI theme={null}
  # Target the project where you want to create the index.
  pc target -o "example-org" -p "example-project"
  # Create the index.
  pc index create \
    --name "standard-sparse-cli" \
    --vector_type "sparse" \
    --metric "dotproduct" \
    --cloud "aws" \
    --region "us-east-1" \
    --tags "environment=development"
  ```
</CodeGroup>

## Create an index from a backup

You can restore an index from a backup, regardless of whether it stores dense or sparse vectors. For more details, see [Restore an index](/guides/manage-data/restore-an-index).

## Metadata indexing

<Warning>
  This feature is in [early access](/release-notes/feature-availability) and available only on the `2025-10` version of the API. The CLI does not yet support this feature.
</Warning>

Pinecone indexes all metadata fields by default. However, large amounts of metadata can cause slower [index building](/guides/get-started/database-architecture#index-builder) as well as slower [query execution](/guides/get-started/database-architecture#query-executors), particularly when data is not cached in a query executor's memory and local SSD and must be fetched from object storage.

To prevent performance issues due to excessive metadata, you can limit metadata indexing to the fields that you plan to use for [query filtering](/guides/search/filter-by-metadata).

### Set metadata indexing

You can set metadata indexing during index creation or [namespace creation](/reference/api/2025-10/data-plane/createnamespace):

* Index-level metadata indexing rules apply to all namespaces that don't have explicit metadata indexing rules.
* Namespace-level metadata indexing rules overrides index-level metadata indexing rules.

For example, let's say you want to store records that represent chunks of a document, with each record containing many metadata fields. Since you plan to use only a few of the metadata fields to filter queries, you would specify the metadata fields to index as follows.

<Warning>
  Metadata indexing cannot be changed after index or namespace creation.
</Warning>

<CodeGroup>
  ```shell Index-level metadata indexing theme={null}
  PINECONE_API_KEY="YOUR_API_KEY"

  curl "https://api.pinecone.io/indexes" \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -H "Api-Key: $PINECONE_API_KEY" \
    -H "X-Pinecone-Api-Version: 2025-10" \
    -d '{
          "name": "example-index-metadata",
          "vector_type": "dense",
          "dimension": 1536,
          "metric": "cosine",
          "spec": {
            "serverless": {
              "cloud": "aws",
              "region": "us-east-1",
              "schema": {
                "fields": {
                  "document_id": {
                    "filterable": true
                  },
                  "document_title": {
                    "filterable": true
                  },
                  "chunk_number": {
                    "filterable": true
                  },
                  "document_url": {
                    "filterable": true
                  },
                  "created_at": {
                    "filterable": true
                  }
                }
              }
            }
          },
          "deletion_protection": "disabled"
        }'
  ```

  ```shell Namespace-level metadata indexing theme={null}
  # To learn how to get the unique host for an index,
  # see https://docs.pinecone.io/guides/manage-data/target-an-index
  PINECONE_API_KEY="YOUR_API_KEY"
  INDEX_HOST="INDEX_HOST"

  curl "https://$INDEX_HOST/namespaces" \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -H "Api-Key: $PINECONE_API_KEY" \
    -H "X-Pinecone-Api-Version: 2025-10" \
    -d '{
          "name": "example-namespace",
          "schema": {
            "fields": {
              "document_id": {
                "filterable": true
              },
              "document_title": {
                "filterable": true
              },
              "chunk_number": {
                "filterable": true
              },
              "document_url": {
                "filterable": true
              },
              "created_at": {
                "filterable": true
              }
            }
          }
        }'
  ```
</CodeGroup>

### Check metadata indexing

To check which metadata fields are indexed, you can describe the index or namespace:

<CodeGroup>
  ```shell Describe index theme={null}
  PINECONE_API_KEY="YOUR_API_KEY"

  curl -X GET "https://api.pinecone.io/indexes/example-index-metadata" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "X-Pinecone-Api-Version: 2025-10"
  ```

  ```shell Describe namespace theme={null}
  # To learn how to get the unique host for an index,
  # see https://docs.pinecone.io/guides/manage-data/target-an-index
  PINECONE_API_KEY="YOUR_API_KEY"
  INDEX_HOST="INDEX_HOST"

  curl -X GET "https://$INDEX_HOST/namespaces/example-namespace" \
       -H "Api-Key: $PINECONE_API_KEY" \
       -H "X-Pinecone-Api-Version: 2025-10"
  ```
</CodeGroup>

The response includes the `schema` object with the names of the metadata fields explicitly indexed during index or namespace creation.

<Note>
  The response does not include unindexed metadata fields or metadata fields indexed by default.
</Note>

<CodeGroup>
  ```json Describe index theme={null}
  {
    "id": "751ab850-6e61-4f92-bd23-fa129803d207",
    "vector_type": "dense",
    "name": "example-index",
    "metric": "cosine",
    "dimension": 1536,
    "status": {
      "ready": false,
      "state": "Initializing"
    },
    "host": "example-index-fa77d8e.svc.aped-4627-b74a.pinecone.io",
    "spec": {
      "serverless": {
        "region": "us-east-1",
        "cloud": "aws",
        "read_capacity": {
          "mode": "OnDemand",
          "status": "Ready"
        },
        "schema": {
          "fields": {
            "document_id": {
              "filterable": true
            },
            "document_title": {
              "filterable": true
            },
            "created_at": {
              "filterable": true
            },
            "chunk_number": {
              "filterable": true
            },
            "document_url": {
              "filterable": true
            }
          }
        }
      }
    },
    "deletion_protection": "disabled",
    "tags": null
  }

  ```

  ```json Describe namespace theme={null}
  {
    "name": "example-namespace",
    "record_count": "20000",
    "schema": {
      "fields": {
        "document_title": {
          "filterable": true
        },
        "document_url": {
          "filterable": true
        },
        "chunk_number": {
          "filterable": true
        },
        "document_id": {
          "filterable": true
        },
        "created_at": {
          "filterable": true
        }
      }
    }
  }
  ```
</CodeGroup>

## Index options

### Cloud regions

When creating an index, you must choose the cloud and region where you want the index to be hosted. The following table lists the available public clouds and regions and the plans that support them:

| Cloud   | Region                       | [Supported plans](https://www.pinecone.io/pricing/) | [Availability phase](/release-notes/feature-availability) |
| ------- | ---------------------------- | --------------------------------------------------- | --------------------------------------------------------- |
| `aws`   | `us-east-1` (Virginia)       | Starter, Builder, Standard, Enterprise              | General availability                                      |
| `aws`   | `us-west-2` (Oregon)         | Builder, Standard, Enterprise                       | General availability                                      |
| `aws`   | `eu-west-1` (Ireland)        | Builder, Standard, Enterprise                       | General availability                                      |
| `aws`   | `eu-central-1` (Frankfurt)   | Builder, Standard, Enterprise                       | General availability                                      |
| `aws`   | `ap-southeast-1` (Singapore) | Builder, Standard, Enterprise                       | General availability                                      |
| `gcp`   | `us-central1` (Iowa)         | Builder, Standard, Enterprise                       | General availability                                      |
| `gcp`   | `europe-west4` (Netherlands) | Builder, Standard, Enterprise                       | General availability                                      |
| `azure` | `eastus2` (Virginia)         | Builder, Standard, Enterprise                       | General availability                                      |

The cloud and region cannot be changed after a serverless index is created.

<Note>
  On the Starter plan, you can create serverless indexes in the `us-east-1` region of AWS only. To create indexes in other regions, [upgrade to the Builder, Standard, or Enterprise plan](/guides/organizations/manage-billing/upgrade-billing-plan).
</Note>

### Similarity metrics

When creating an index that stores dense vectors, you can choose from the following similarity metrics. For the most accurate results, choose the similarity metric used to train the embedding model for your vectors. For more information, see [Vector Similarity Explained](https://www.pinecone.io/learn/vector-similarity/).

<Note>Indexes that store [sparse vectors](#sparse-indexes) must use the `dotproduct` metric.</Note>

<AccordionGroup>
  <Accordion title="Euclidean">
    Querying indexes with this metric returns a similarity score equal to the squared Euclidean distance between the result and query vectors.

    This metric calculates the square of the distance between two data points in a plane. It is one of the most commonly used distance metrics. For an example, see our [IT threat detection example](https://colab.research.google.com/github/pinecone-io/examples/blob/master/docs/it-threat-detection.ipynb).

    When you use `metric='euclidean'`, the most similar results are those with the **lowest similarity score**.
  </Accordion>

  <Accordion title="Cosine">
    This is often used to find similarities between different documents. The advantage is that the scores are normalized to \[-1,1] range. For an example, see our [generative question answering example](https://colab.research.google.com/github/pinecone-io/examples/blob/master/docs/gen-qa-openai.ipynb).
  </Accordion>

  <Accordion title="Dotproduct">
    This is used to multiply two vectors. You can use it to tell us how similar the two vectors are. The more positive the answer is, the closer the two vectors are in terms of their directions. For an example, see our [semantic search example](https://colab.research.google.com/github/pinecone-io/examples/blob/master/docs/semantic-search.ipynb).
  </Accordion>
</AccordionGroup>

### Embedding models

[Dense vectors](/guides/get-started/concepts#dense-vector) and [sparse vectors](/guides/get-started/concepts#sparse-vector) are the basic units of data in Pinecone and what Pinecone was specially designed to store and work with. Dense vectors represents the semantics of data such as text, images, and audio recordings, while sparse vectors represent documents or queries in a way that captures keyword information.

To transform data into vector format, you use an embedding model. Pinecone hosts several embedding models so it's easy to manage your vector storage and search process on a single platform. You can use a hosted model to embed your data as an integrated part of upserting and querying, or you can use a hosted model to embed your data as a standalone operation.

The following embedding models are hosted by Pinecone.

<Note>
  To understand how cost is calculated for embedding, see [Embedding cost](/guides/manage-cost/understanding-cost#embedding). To get model details via the API, see [List models](/reference/api/latest/inference/list_models) and [Describe a model](/reference/api/latest/inference/describe_model).
</Note>

#### multilingual-e5-large

[`multilingual-e5-large`](/models/multilingual-e5-large) is an efficient dense embedding model trained on a mixture of multilingual datasets. It works well on messy data and short queries expected to return medium-length passages of text (1-2 paragraphs).

**Details**

* Vector type: Dense
* Modality: Text
* Dimension: 1024
* Recommended similarity metric: Cosine
* Max sequence length: 507 tokens
* Max batch size: 96 sequences

For rate limits, see [Embedding tokens per minute](/reference/api/database-limits#embedding-tokens-per-minute-per-model) and [Embedding tokens per month](/reference/api/database-limits#embedding-tokens-per-month-per-model).

**Parameters**

The `multilingual-e5-large` model supports the following parameters:

| Parameter    | Type   | Required/Optional | Description                                                                                                                                                                                                                                    | Default |
| :----------- | :----- | :---------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------ |
| `input_type` | string | Required          | The type of input data. Accepted values: `query` or `passage`.                                                                                                                                                                                 |         |
| `truncate`   | string | Optional          | How to handle inputs longer than those supported by the model. Accepted values: `END` or `NONE`.<br /><br />`END` truncates the input sequence at the input token limit. `NONE` returns an error when the input exceeds the input token limit. | `END`   |

#### llama-text-embed-v2

[`llama-text-embed-v2`](/models/llama-text-embed-v2) is a high-performance dense embedding model optimized for text retrieval and ranking tasks. It is trained on a diverse range of text corpora and provides strong performance on longer passages and structured documents.

**Details**

* Vector type: Dense
* Modality: Text
* Dimension: 1024 (default), 2048, 768, 512, 384
* Recommended similarity metric: Cosine
* Max sequence length: 2048 tokens
* Max batch size: 96 sequences

For rate limits, see [Embedding tokens per minute](/reference/api/database-limits#embedding-tokens-per-minute-per-model) and [Embedding tokens per month](/reference/api/database-limits#embedding-tokens-per-month-per-model).

**Parameters**

The `llama-text-embed-v2` model supports the following parameters:

| Parameter    | Type    | Required/Optional | Description                                                                                                                                                                                                                                    | Default |
| :----------- | :------ | :---------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------ |
| `input_type` | string  | Required          | The type of input data. Accepted values: `query` or `passage`.                                                                                                                                                                                 |         |
| `truncate`   | string  | Optional          | How to handle inputs longer than those supported by the model. Accepted values: `END` or `NONE`.<br /><br />`END` truncates the input sequence at the input token limit. `NONE` returns an error when the input exceeds the input token limit. | `END`   |
| `dimension`  | integer | Optional          | Dimension of the vector to return.                                                                                                                                                                                                             | 1024    |

#### pinecone-sparse-english-v0

[`pinecone-sparse-english-v0`](/models/pinecone-sparse-english-v0) is a sparse embedding model for converting text to [sparse vectors](/guides/get-started/concepts#sparse-vector) for [sparse-vector lexical search](/guides/search/lexical-search) or hybrid search. Built on the innovations of the [DeepImpact architecture](https://arxiv.org/pdf/2104.12016), the model directly estimates the lexical importance of tokens by leveraging their context, unlike traditional retrieval models like BM25, which rely solely on term frequency.

**Details**

* Vector type: Sparse
* Modality: Text
* Recommended similarity metric: Dotproduct
* Max sequence length: 512 or 2048
* Max batch size: 96 sequences

For rate limits, see [Embedding tokens per minute](/reference/api/database-limits#embedding-tokens-per-minute-per-model) and [Embedding tokens per month](/reference/api/database-limits#embedding-tokens-per-month-per-model).

**Parameters**

The `pinecone-sparse-english-v0` model supports the following parameters:

| Parameter                 | Type    | Required/Optional | Description                                                                                                                                                                                                                                                                    | Default |
| :------------------------ | :------ | :---------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------ |
| `input_type`              | string  | Required          | The type of input data. Accepted values: `query` or `passage`.                                                                                                                                                                                                                 |         |
| `max_tokens_per_sequence` | integer | Optional          | Maximum number of tokens to embed. Accepted values: `512` or `2048`.                                                                                                                                                                                                           | `512`   |
| `truncate`                | string  | Optional          | How to handle inputs longer than those supported by the model. Accepted values: `END` or `NONE`.<br /><br />`END` truncates the input sequence at the the `max_tokens_per_sequence` limit. `NONE` returns an error when the input exceeds the `max_tokens_per_sequence` limit. | `END`   |
| `return_tokens`           | boolean | Optional          | Whether to return the string tokens.                                                                                                                                                                                                                                           | `false` |
