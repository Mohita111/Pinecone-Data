> ## Documentation Index
> Fetch the complete documentation index at: https://docs.pinecone.io/llms.txt
> Use this file to discover all available pages before exploring further.

# Architecture

> Learn how Pinecone's architecture enables fast, relevant vector search at any scale.

## Overview

Pinecone runs as a managed service on AWS, GCP, and Azure cloud platforms. When you send a request to Pinecone, it goes through an [API gateway](#api-gateway) that routes it to either a global [control plane](#control-plane) or a regional [data plane](#data-plane). All your vector data is stored in highly efficient, distributed [object storage](#object-storage).

<img className="block max-w-full dark:hidden" noZoom src="https://mintcdn.com/pinecone/HsZKO51bNmpAasdT/images/serverless-overview.svg?fit=max&auto=format&n=HsZKO51bNmpAasdT&q=85&s=a61fd08e854c5d3d660891ba33bee38a" width="740" height="360" data-path="images/serverless-overview.svg" />

<img className="hidden max-w-full dark:block" noZoom src="https://mintcdn.com/pinecone/HsZKO51bNmpAasdT/images/serverless-overview-dark.svg?fit=max&auto=format&n=HsZKO51bNmpAasdT&q=85&s=abb1082e778c81cec11052add4fcdb9e" width="740" height="360" data-path="images/serverless-overview-dark.svg" />

### API gateway

Every request to Pinecone includes an [API key](/guides/projects/manage-api-keys) that's assigned to a specific [project](/guides/projects/understanding-projects). The API gateway first validates your API key to make sure you have permission to access the project. Once validated, it routes your request to either the global control plane (for managing projects and indexes) or a regional data plane (for reading and writing data), depending on what you're trying to do.

### Control plane

The global control plane manages your organizational resources like projects and indexes. It uses a dedicated database to keep track of all these objects. The control plane also handles billing, user management, and coordinates operations across different regions.

### Data plane

The data plane handles all requests to write and read records in [indexes](/guides/index-data/indexing-overview) within a specific [cloud region](/guides/index-data/create-an-index#cloud-regions). Each index is divided into one or more logical [namespaces](/guides/index-data/indexing-overview#namespaces), and all your data read and write requests target a specific namespace.

Pinecone separates write and read operations into different paths, with each scaling independently based on demand. This separation ensures that your queries never slow down your writes, and your writes never slow down your queries.

### Object storage

For each namespace in a serverless index, Pinecone organizes records into immutable files called slabs. These slabs are [optimized for fast querying](#index-builder) and stored in distributed object storage that provides virtually unlimited scalability and high availability.

## Write path

<img className="block max-w-full dark:hidden" noZoom src="https://mintcdn.com/pinecone/HsZKO51bNmpAasdT/images/serverless-write-path.svg?fit=max&auto=format&n=HsZKO51bNmpAasdT&q=85&s=0f9d9fde37626e8b06a537dd62cf2a3b" width="940" height="620" data-path="images/serverless-write-path.svg" />

<img className="hidden max-w-full dark:block" noZoom src="https://mintcdn.com/pinecone/HsZKO51bNmpAasdT/images/serverless-write-path-dark.svg?fit=max&auto=format&n=HsZKO51bNmpAasdT&q=85&s=d4d211a1a0ffbaae94685c327b434c26" width="940" height="620" data-path="images/serverless-write-path-dark.svg" />

### Request log

When you send a write request (to add, update, or delete records), the [data plane](#data-plane) first logs the request details with a unique sequence number (LSN). This ensures all operations happen in the correct order and provides a way to track the state of the index.

Pinecone immediately returns a `200 OK` response, guaranteeing that your write is durable and won't be lost. The system then processes your write in the background.

### Index builder

The index builder stores your write data in an in-memory structure called a memtable. This includes your vector data, any metadata you've attached, and the sequence number. If you're updating or deleting a record, the system also tracks how to handle the old version during queries.

Periodically, the index builder moves data from the memtable to permanent storage. In [object storage](#object-storage), your data is organized into immutable files called slabs. These slabs are optimized for query performance. Smaller slabs use fast indexing techniques that provide good performance with minimal resource requirements. As slabs grow, the system merges them into larger slabs that use more sophisticated methods that provide better performance at scale. This adaptive process both optimizes query performance for each slab and amortizes the cost of more expensive indexing through the lifetime of the namespace.

<Note>
  All read operations check the memtable first, so you can immediately search data that you've just written, even before it's moved to permanent storage. For more details, see [Query executors](#query-executors).
</Note>

## Read path

<img className="block max-w-full dark:hidden" noZoom src="https://mintcdn.com/pinecone/HsZKO51bNmpAasdT/images/serverless-read-path.svg?fit=max&auto=format&n=HsZKO51bNmpAasdT&q=85&s=144df73e8dc7ad64eb60c0e03ce86c65" width="780" height="620" data-path="images/serverless-read-path.svg" />

<img className="hidden max-w-full dark:block" noZoom src="https://mintcdn.com/pinecone/HsZKO51bNmpAasdT/images/serverless-read-path-dark.svg?fit=max&auto=format&n=HsZKO51bNmpAasdT&q=85&s=0f808f3a7b7200064ad7312ccd818dd0" width="780" height="620" data-path="images/serverless-read-path-dark.svg" />

### Query routers

When you send a search query, the [data plane](#data-plane) first validates your request and checks that it meets system limits like [rate and object limits](/reference/api/database-limits). The query router then identifies which slabs contain relevant data and routes your query to the appropriate executors. It also searches the memtable for any recent data that hasn't been moved to permanent storage yet.

### Query executors

Each query executor searches through its assigned slabs and returns the most relevant candidates to the query router. If your query includes metadata filters, the executors exclude records that don't match your criteria before finding the best matches.

Most of the time, the slabs are cached in memory or on local SSD, which provides very fast query performance. If a slab isn't cached (which happens when it's accessed for the first time or hasn't been used recently), the executor fetches it from object storage and caches it for future queries.

The query router then combines results from all executors, removes duplicates, merges them with results from the memtable, and returns the final set of best matches to you.
