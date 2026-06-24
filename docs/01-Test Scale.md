> ## Documentation Index
> Fetch the complete documentation index at: https://docs.pinecone.io/llms.txt
> Use this file to discover all available pages before exploring further.

# Test Pinecone at scale

> Test Pinecone with a real-world dataset and semantic search workload.

This guide walks you through testing Pinecone at production scale. You'll import 10 million vectors, run a benchmark, and analyze the results to verify Pinecone meets production requirements for semantic search applications.

<Note>
  This test requires a Pinecone account on the Standard or Enterprise plan because it uses [import from object storage](/guides/index-data/import-data), which is not available on the Starter or Builder plans. New users can sign up for the [Standard trial](/guides/organizations/manage-billing/standard-trial) for 21 days and \$300 in credits, more than enough to cover the costs of this test. Existing users on the Starter or Builder plan can [upgrade](/guides/organizations/manage-billing/upgrade-billing-plan).
</Note>

## About this test

Semantic search enables finding relevant content based on meaning rather than exact keyword matches, making it ideal for applications like product search, content recommendation, and question-answering systems. This test simulates a production-scale semantic search workload, measuring import time, query throughput, query latency, and associated costs.

The test uses the following configuration:

* **Records**: 10 million records from the [Amazon Reviews 2023](https://amazon-reviews-2023.github.io/) dataset
* **Embedding model**: `llama-text-embed-v2` (1024 dimensions)
* **Similarity metric**: cosine
* **Total size**: 48.8 GB
* **Query load**: 10 queries per second total (across all users)
* **Concurrent users**: 10 users querying simultaneously
* **Test queries**: 100,000 queries
* **Import time target**: \< 30 minutes
* **Query latency target**: p90 latency \< 100ms

**Estimated cost**: \~\$127 (import: \$48.80, queries: \$78.08, storage: \$0.09) — see [detailed cost breakdown](#6-check-costs)

## 1. Get an API key

To follow the steps in this guide, you'll need an API key. Create a new API key in the [Pinecone console](https://app.pinecone.io/organizations/-/keys), or use this widget:

<div style={{minWidth: '450px', minHeight:'152px'}}>
  <div id="pinecone-connect-widget">
    <div class="connect-widget-skeleton">
      <div class="skeleton-content" />
    </div>
  </div>
</div>

Your generated API key:

```shell theme={null}
"{{YOUR_API_KEY}}"
```

## 2. Create an index

<Note>
  This test requires you to use AWS-based indexes and infrastructure. The sample dataset is only available from Amazon S3, and you can only import from Amazon S3 to Pinecone indexes hosted on AWS. To run the benchmark, you'll need to provision an AWS EC2 instance in the same region as your index.
</Note>

Create an on-demand index that matches the dimensions and similarity metric of the dataset you'll import in later steps.

<Tabs>
  <Tab title="Console">
    1. In the Pinecone console, go to the [Indexes](https://app.pinecone.io/organizations/-/projects/-/indexes) page.
    2. Click **Create index**.
    3. Check **Custom settings**.
    4. Configure the index with the following settings:
       * **Name**: `search-10m`
       * **Vector type**: Dense
       * **Dimensions**: `1024`
       * **Metric**: cosine
       * **Capacity mode**: Serverless (on-demand)
       * **Cloud**: AWS (required for this test)
       * **Region**: Use an AWS region appropriate for your use case (for example, `us-east-1`)
    5. Click **Create index**.
  </Tab>

  <Tab title="Code">
    If using code to create an index, first install the [Python SDK](/reference/python-sdk):

    ```shell Terminal theme={null}
    pip install pinecone
    ```

    Then, create the index:

    <CodeGroup>
      ```python Python theme={null}
      from pinecone import Pinecone, ServerlessSpec

      pc = Pinecone(api_key="{{YOUR_API_KEY}}")

      index_name = "search-10m"

      if not pc.has_index(index_name):
          pc.create_index(
              name=index_name,
              vector_type="dense",
              dimension=1024,
              metric="cosine",
              spec=ServerlessSpec(
                  # AWS is required for this test
                  cloud="aws",   
                  # Use an AWS region appropriate for your use case
                  region="us-east-1" 
              )
          )
      ```
    </CodeGroup>
  </Tab>
</Tabs>

## 3. Import the dataset

Pinecone's [import feature](/guides/index-data/import-data) enables you to load millions of vectors from object storage in parallel. In this step, you'll import 10 million records into a single namespace (`ns_2`) in your index.

### Choose an import source

To import the dataset, you'll need to use the following Amazon S3 import URL:

```
s3://fe-customer-pocs/search/search_10M/dense/
```

### Start and monitor the import

For this dataset, the import should take less than 30 minutes.

<Tabs>
  <Tab title="Console">
    1. In the Pinecone console, go to the [Indexes](https://app.pinecone.io/organizations/-/projects/-/indexes) page.
    2. Find your `search-10m` index and click **... > Import data**.
    3. For **Storage integration**, select **No integration (public bucket)**.
    4. Enter the import URL: `s3://fe-customer-pocs/search/search_10M/dense/`.
    5. For **Error handling**, select **Abort on error (default)**.
    6. Click **Start import**.

    To monitor progress, open your index in the Pinecone console and navigate to the **Imports** tab. After the import completes, compare the **Started time** and **End time** timestamps to see the total time required.

    <Note>
      For this dataset, the import should take around 30 minutes. While the import is running, you can continue with the next step to provision a VM and install VSB. However, wait for the import to complete before running the benchmark.
    </Note>
  </Tab>

  <Tab title="Code">
    Start the import:

    <CodeGroup>
      ```python Python theme={null}
      from pinecone import Pinecone, ImportErrorMode

      pc = Pinecone(api_key="{{YOUR_API_KEY}}")
      index = pc.Index("search-10m")

      import_response = index.start_import(
          uri="s3://fe-customer-pocs/search/search_10M/dense/",
          error_mode=ImportErrorMode.ABORT 
      )
      print(f"Import started: {import_response['id']}")
      ```
    </CodeGroup>

    Monitor import progress:

    <CodeGroup>
      ```python Python theme={null}
      from pinecone import Pinecone
      import time 

      pc = Pinecone(api_key="{{YOUR_API_KEY}}")
      index = pc.Index("search-10m")

      while True:
          status = index.describe_import(id="IMPORT_ID")
          print(f"Status: {status['status']}, Progress: {status['percent_complete']:.1f}%")
          if status['status'] == "Completed":
              print("Import completed successfully!")
              break
          elif status['status'] == "Failed":
              print("Import failed. Check error details.")
              break
          elif status['status'] == "Cancelled":
              print("Import cancelled.")
              break
          time.sleep(15) # Check every 15 seconds
      ```
    </CodeGroup>

    <Note>
      There are three ways to find the import ID:

      * It's returned when the import is started
      * In the Pinecone console, on the **Imports** tab for your index
      * By calling [List imports](/reference/api/latest/data-plane/list_imports)
    </Note>
  </Tab>
</Tabs>

## 4. Run the benchmark

To simulate realistic query patterns and measure latency and throughput for your Pinecone index, use [Vector Search Bench (VSB)](https://github.com/pinecone-io/VSB). The benchmark runs 100,000 queries at 10 queries per second, which should take just under three hours to complete.

<Steps>
  <Step title="Provision a VM">
    VSB reports latency as the time from when the tool issues a query to when the query is returned by Pinecone.

    To minimize the client-side latency between the tool and Pinecone, run the benchmark on a dedicated AWS EC2 instance that's hosted in the same AWS region as your Pinecone index. This reduces the client-side latency to sub-millisecond range.

    <Note>
      As noted in [section 2](#2-create-an-index), this test requires an AWS EC2 instance in the same region as your index.
    </Note>

    For instructions on how to provision an EC2 instance, see the [AWS documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/LaunchingAndUsingInstances.html).

    <Note>
      Create a VM that comes with Python 3.11 or higher.
    </Note>
  </Step>

  <Step title="Connect to the VM">
    Connect to the VM using SSH or the cloud provider's console.
  </Step>

  <Step title="Install Vector Search Bench (VSB)">
    [VSB (Vector Search Bench)](https://github.com/pinecone-io/VSB) is a benchmarking suite for testing vector database search performance across different workloads and databases. To install it, you'll first need to install various dependencies.

    1. **Verify Python version**

       VSB requires Python 3.11 or higher to run. Verify your Python version:

       ```bash Terminal theme={null}
       python3 --version
       ```

       If your version is below 3.11, install Python 3.11+ using your distribution's package manager.

    2. **Install git**

       Git is required to clone the VSB repository. Check if git is installed:

       ```bash Terminal theme={null}
       git --version
       ```

       If git is not installed, install it using your system's package manager:

       ```bash Terminal theme={null}
       # Adapt for your VM's package manager (apt/yum/dnf)
       sudo apt-get update && sudo apt-get install git
       ```

    3. **Install pipx**

       pipx is required to install Poetry. First, check if pip3 is installed:

       ```bash Terminal theme={null}
       pip3 --version
       ```

       If pip is not installed, install it using your system's package manager:

       ```bash Terminal theme={null}
       # Adapt for your VM's package manager (apt/yum/dnf)
       sudo apt-get update && sudo apt-get install python3-pip
       ```

       Then check if pipx is installed:

       ```bash Terminal theme={null}
       pipx --version
       ```

       If pipx is not installed, install it via your system's package manager:

       ```bash Terminal theme={null}
       # Adapt for your VM's package manager (apt/yum/dnf)
       sudo apt-get update && sudo apt-get install pipx
       pipx ensurepath
       ```

       After installation, run this command to update the PATH in your current terminal session:

       ```bash Terminal theme={null}
       source ~/.bashrc
       ```

    4. **Install Poetry**

       [Poetry](https://python-poetry.org/) is required to manage [VSB's](https://github.com/pinecone-io/VSB) Python dependencies and virtual environment. If Poetry is not [installed](https://python-poetry.org/docs/), use pipx to install it:

       ```bash Terminal theme={null}
       pipx install poetry
       ```

       Alternatively, use the [official Poetry installer](https://python-poetry.org/docs/#installing-with-the-official-installer).

    5. **Clone the VSB repository**

       To run the benchmark, you'll first need to clone the VSB repository and navigate to it:

       ```bash Terminal theme={null}
       git clone https://github.com/pinecone-io/VSB.git
       cd VSB
       ```

    6. **Configure Poetry**

       Since your VM has Python 3.11 or higher installed (as specified in the VM provisioning step), tell Poetry to use it:

       ```bash Terminal theme={null}
       poetry env use python3
       ```

    7. **Install dependencies**

       VSB requires several Python packages to run. Install all dependencies:

       ```bash Terminal theme={null}
       poetry install
       ```
  </Step>

  <Step title="Benchmark your Pinecone index">
    To test the performance of your Pinecone index, run the following command from within the `VSB` directory. For more information about VSB, see its [GitHub repository](https://github.com/pinecone-io/VSB).

    The following command simulates 10 concurrent users issuing a total of 100,000 queries at 10 queries per second (QPS). Each query performs a vector search for the top 10 most similar 1024-dimensional vectors, using cosine similarity, with query vectors selected uniformly at random. The `--skip_populate` flag skips the data population phase, since you've already imported data into your index.

    ```bash Terminal theme={null}
    poetry run vsb \
        --database="pinecone" \
        --workload=synthetic-proportional \
        --pinecone_api_key="{{YOUR_API_KEY}}" \
        --pinecone_index_name="search-10m" \
        --pinecone_namespace_name="ns_2" \
        --synthetic_dimensions=1024 \
        --synthetic_metric=cosine \
        --synthetic_top_k=10 \
        --synthetic_requests=100000 \
        --users=10 \
        --requests_per_sec=10 \
        --synthetic_query_distribution=uniform \
        --synthetic_query_ratio=1 \
        --synthetic_insert_ratio=0 \
        --synthetic_delete_ratio=0 \
        --synthetic_update_ratio=0 \
        --skip_populate
    ```
  </Step>
</Steps>

## 5. Analyze performance

At the end of the run, VSB prints an operation summary including the requests per second achieved and latencies at different percentiles. Here's an example output:

```shell Terminal theme={null}
2025-12-23T00:34:37 INFO     Completed Run phase, took 9940.14s

                      Operation Summary

  Operation  Requests  Failures  Requests/sec  Failures/sec
 ───────────────────────────────────────────────────────────
  Search        99000     0(0%)            10           0.0

                                                    Metrics Summary

  Operation  Metric         Min  0.1%    1%    5%   10%   25%   50%   75%   90%   95%   99%  99.9%  99.99%   Max  Mean
 ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  Search     Latency (ms)    23    25    25    26    27    27    29    34    44    81   350    430    1300  4602    43
```

Confirm that the requests per second achieved is around 10 QPS and the p90 latency is less than 100ms.

To see more detailed statistics, you can analyze the `stats.json` file identified in the output.

## 6. Check costs

You can check the costs for the import, queries, and storage in the Pinecone console at [Settings > Usage](https://app.pinecone.io/organizations/-/settings/usage). Cost data is delayed up to three days, but once it's available, compare the actual costs to the estimated costs below.

<Note>For the latest pricing details, see [Pricing](https://www.pinecone.io/pricing).</Note>

| Cost type | Amount          | Pricing                | Estimated cost |
| :-------- | :-------------- | :--------------------- | :------------- |
| Import    | 48.8 GB         | \$0.25/GB              | \$12.20        |
| Queries   | 100,000 queries | \$16 per 1M read units | \$78.08        |
| Storage   | 4 hours         | \$0.33/GB/month        | \$0.09         |
| **Total** |                 |                        | **\$90.37**    |

<Note>Standard and Enterprise organizations receive a **one-time 1 TB bulk import credit**, valid through July 31, 2026, so on those plans the import portion of this example would be \$0 and the total would be \$78.17. The figures above assume the standard overage rate.</Note>

<Steps>
  <Step title="Import costs">
    The current price for import is \$0.25/GB. The dataset size for this test is 48.8 GB, so the import cost should be \$12.20.
  </Step>

  <Step title="Query costs">
    A query uses 1 [read unit (RU)](/guides/manage-cost/understanding-cost#read-units) for every 1 GB of namespace size. The current price for queries in the `us-east-1` region of AWS is \$16 per 1 million read units (pricing varies by region).

    This test ran 100,000 queries against a namespace size of 48.8 GB. Each query uses 48.8 RUs (1 RU per GB), so the total is 4,880,000 RUs. At \$16 per 1 million RUs, the cost is (4,880,000 / 1,000,000) × \$16 = \$78.08.
  </Step>

  <Step title="Storage costs">
    The current price for storage is \$0.33 per GB per month. The dataset size for this test is 48.8 GB. Assuming a total storage time of 4 hours (including import, benchmark runtime, and cleanup), the storage cost is: \$0.33/GB/month \* 48.8 GB / 730 hours \* 4 hours = \$0.09.
  </Step>

  <Step title="Total costs">
    The total cost for the test is the sum of the import cost, query cost, and storage cost: \$12.20 + \$78.08 + \$0.09 = \$90.37.
  </Step>
</Steps>

## 7. Clean up

When you no longer need your test index, [delete it](/guides/manage-data/manage-indexes#delete-an-index) to avoid incurring unnecessary costs.
