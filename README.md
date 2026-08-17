# Explainer: Built-in Vector Store API

This proposal is an early design sketch by the Chrome Built-in AI team to
describe the problem below and solicit feedback on the proposed solution. It has
not been approved to ship in Chrome.

## Proponents

- Chrome Built-in AI Team

## Participate

- <https://github.com/explainers-by-googlers/vector-store-api/issues>


## Introduction

The **Vector Store API** is a proposed Web Platform API that provides an
end-to-end semantic retrieval system entirely on the user's device. By executing
strictly on-device, it delivers significant end-user benefits: it protects user
privacy by keeping sensitive data local, ensures offline availability, and offers
near-instant performance by eliminating network round-trips.

Beyond these end-user benefits, the API significantly lowers the barrier to entry
for web developers. Instead of requiring developers to manually manage vectors,
the API automatically handles data ingestion, text chunking, embedding generation,
indexing, and semantic similarity search. By providing a "managed" semantic search
experience directly in the browser—similar to high-level industry APIs (e.g.,
OpenAI Vector Store API)—this high-level abstraction handles low-level data
orchestration and vector mechanics natively.

## Goals

- **Privacy-First Design**: The embeddings are generated locally and remain
entirely on-device, enabling developers to process and search sensitive user
data privately.

- **Low Latency**: Enable real-time semantic search and retrieval by
eliminating network round-trips for both embedding generation and vector
similarity matching.

- **Maximum API Simplicity**: Provide a high-level, document-based JavaScript
interface. Developers interact only with strings (text)/similarity scores and
never have to manage or manipulate high-dimensional float arrays (raw
embeddings).

- **Automatic Storage Management**: Provide a built-in mechanism for persisting
the vectors, removing the need for developers to build DIY vector indexes in
IndexedDB or OPFS.

- **Cross-Browser Interoperability:** By encapsulating the vector generation and
search processes, the API provides a universal text-in, text-out interface.
Developers do not need to worry about different browser vendors shipping
underlying embedding models with mathematically incompatible vector spaces, as
long as the embedding generation and vector search happen entirely on-device.

## Non-goals

- **Integration with External Vector Ecosystems**: Because different browser
vendors may utilize distinct underlying embedding models, the resulting vector
spaces will be mathematically incompatible across implementations. Relying on
these raw embeddings for external architectures—such as syncing client-generated
vectors with a server-side database—would introduce cross-browser
interoperability fragmentation. Until we identify an acceptable solution to this
challenge, this initial version of the API focuses strictly on self-contained,
on-device loops where indexing and retrieval occur within the same client
environment. Accordingly, the API currently encapsulates raw embeddings
specifically to prevent developers from building premature, browser-dependent
architectures.

- **Model Training**: This API is for inference and indexing only. It does not
support training or fine-tuning models on the device.

## Use Cases

**Semantic Search**

A note-taking web app could offer "semantic search" to find notes based on
meaning, not just keywords, entirely on-device and private to the user.

**Retrieval-Augmented Generation (RAG)**

A documentation site could build a simple, offline-capable Q\&A bot (RAG) that
finds the most relevant passages to answer a user's question.

**Real-time Content Intelligence**

A small, community-run forum could offer real-time, on-device moderation hints,
flagging potentially toxic comments as the user is typing, before the content is
ever sent to a server.

## Potential Solution

We propose a new interface, VectorStore, which provides an end-to-end semantic
retrieval system. The API follows the pattern established by other built-in AI
APIs, namely the sequence of availability -\> create -\> execute function calls.

### Basic Usage: Create, Insert, and Search

Developers interact with a VectorStore object. By passing raw text strings into
the insert() method, the browser takes over the complexity of preparing the
data: it automatically chunks the text according to the store's configuration,
generates the embeddings using the on-device model, and indexes the results into
a local database.

```javascript
// 1. Check if the API is available
if (!VectorStore || (await VectorStore.availability()) === "unavailable") {
  console.error("Vector Store API is not available on this device.");
  return;
}

// 2. Create a vector store with configuration.
// Note: Developers can query `await VectorStore.params()` to discover
// the model's maximum supported chunk size and overlap.
const vectorStore = await VectorStore.create({
  id: "my-app-notes",
  taskType: "retrieval", // Automatically applies task-specific embeddings
  chunkingStrategy: {
    // Browsers select safe 'middle-of-the-road' default values.
    maxChunkSizeTokens: 400, // The maximum number of tokens in each chunk
    chunkOverlapTokens: 50 // The number of tokens that overlap between chunks
  },
  distanceType: 'Cosine'
});

// 3. Insert raw text strings.
// The browser automatically chunks the text according to the strategy defined
// above, generates embeddings using the corresponding task type (e.g., Retrieval (Document)),
// and indexes them under the hood.
const insertResult = await vectorStore.insert(
[
  {
    // unique id is auto-generated if not provided.
    id: "note_123",
    content: "The quick brown fox jumps over the lazy dog.....",
    // This list could be expanded by browser to support new features, e.g.,
    // classification.
  },
  {
    id: "note_456",
    content: "Built-in AI APIs use on-device models.",
  }
]);
console.log(insertResult);
/*
{
  allResponses: [],
  hasErrors: false,
  contentIds: [note_123, note_456]
  insertedCount: 2
}
*/

// 4. Perform a semantic search using text.
// The query string is automatically embedded using the corresponding query task type (e.g., Retrieval (Query)).
const searchResults = await vectorStore.findNearest("fox, the animal", {
  maxNumResults: 1,
  scoreThreshold: 0.5
});

// 5. Review results.
console.log(searchResults);
/*
[
  {
    score: 0.89,
    content: "The quick brown fox jumps over the lazy dog....."
  }
]
*/

// 6. Proactively close the connection to release resources
vectorStore.close();
```

### Listing, Retrieving and Deleting Existing Vector Stores

Because vector stores persistently index data on the user's device, developers
can list available databases and retrieve previously created stores in future
sessions. When retrieving an existing store, developers do not need to redefine
its configuration (like the chunking strategy) or re-ingest the documents.

```javascript
// 1. Fetch the list of all vector stores available to this origin.
const availableStores = await VectorStore.list();
console.log(availableStores);
/*
[
  {
    id: "my-app-notes",
    taskType: "retrieval",
    createdAt: 1718293849392,
    lastModified: 1718293999123,
    itemCount: 42,
    configuration: {
      maxChunkSizeTokens: 400,
      chunkOverlapTokens: 50
    }
  },
  {
    id: "offline-docs-db"
    createdAt: 1718293849392,
    lastModified: 1718293999123,
    itemCount: 21,
    configuration: {
      maxChunkSizeTokens: 300,
      chunkOverlapTokens: 20
    }
  }
]
*/

// 2. Retrieve the vector store by id.
// Notice that we do not pass the `chunkingStrategy` here, as those
// structural settings are inherited from when the store was created.
const vectorStore = await VectorStore.retrieve("my-app-notes");
console.log(vectorStore.metadata);
/*
{
  id: "my-app-notes",
  taskType: "retrieval",
  createdAt: 1718293849392,
  lastModified: 1718293999123,
  itemCount: 42,
  configuration: {
    maxChunkSizeTokens: 400,
    chunkOverlapTokens: 50
  }
}
*/

// 3. Delete the vector store if it's no longer needed.
await VectorStore.delete("my-app-notes");
```

### Listing, Updating & Deleting Contents inside Existing Vector Stores

To support incremental updates without forcing developers to rebuild the entire
vector store, the API provides content-level modification methods. Because
chunking boundaries can shift if text is edited, updates are handled strictly at
the content level.

```javascript
const vectorStore = await VectorStore.retrieve("my-app-notes");
// 1. List contents
const contents = await vectorStore.listContents();
console.log(contents);
/*
[
  {
    id: "note_123",
    vectorStoreId: "my-app-notes",
    createdAt: 1718293849392,
    lastModified: 1718293999123,
  },
  {
    id: "note_456",
    vectorStoreId: "my-app-notes",
    createdAt: 1718293849392,
    lastModified: 1718293999123,
  },
]
*/

// 2. Update a content.
await vectorStore.updateContent("note_123", "Here's the new content");
// 3. Delete a content.
await vectorStore.deleteContent("note_123");
```

### How this solution would solve the use cases

While the core mechanics of creating a vector store, inserting text, and
searching remain identical across these scenarios, the flexibility of the Vector
Store API allows it to power entirely different user experiences.

**Retrieval-Augmented Generation (RAG)**

In this scenario, a developer combines the Vector Store API (for retrieval) with
the built-in Prompt API (for text generation) to create an offline-capable Q\&A
bot.

```javascript
// 1. Retrieve the pre-indexed vector store containing the documentation.
const vectorStore = await VectorStore.retrieve("offline-docs-db");
// 2. The user types a question into the chat UI.
const userQuestion = "How do I configure the main router?";
// 3. Query the Vector Store to find the most relevant passages.
const searchResults = await vectorStore.findNearest(userQuestion, {
  maxNumResults: 3 });
// 4. Extract the text content from the search results to build the context.
const context = searchResults.map(result =\> result.content).join("\\n\\n");
// 5. Use the Prompt API (another Built-in AI API) to synthesize an answer.
const session = await LanguageModel.create({
  initialPrompts: [
    {
      role: "system",
      content: "You are a helpful assistant. Answer the user's question using ONLY the provided context. Context:\n" + context
    }
  ]
});
// 6. The synthesized answer is generated entirely on-device and displayed to the user.
const answer = await session.prompt(userQuestion);
console.log(answer);
```

**Semantic Search**

For the semantic search use case in a note-taking app, the developer would use
the exact same retrieval logic shown in steps 1 through 3 above. However,
instead of passing the results to a language model for synthesis, they would
simply render the retrieved notes directly into the application's search results
UI, allowing the user to find notes conceptually rather than by exact keyword.

**Real-time Content Intelligence**

For real-time moderation, the developer would pre-index a vector store with
examples of known toxic or unhelpful comments. As the user types their post, the
application would continuously query the vector store with the user's draft
text. If the vectorStore.findNearest() returns a match with a high similarity
score, the application can instantly flag the comment with a client-side warning
before it is ever submitted to the server.

## Detailed Design Discussion

### Task Type

The `taskType` option allows developers to specify the intended use case for the vector store, such as `retrieval` or `question_answering`. By defining a task type during creation:
- **Validation**: The API validates whether the requested task type is supported by the underlying embedding model, failing early if it is not.
- **Automatic Contextual Embedding**: The vector store automatically applies the appropriate contextual embedding based on the operation. For example, when using `taskType: "retrieval"`, the API automatically applies the "Retrieval (Document)" task type when embedding text during `insert()`, and applies the "Retrieval (Query)" task type when embedding the query string during `findNearest()`.
- **Store-Level Configuration**: The `taskType` is intentionally defined once during vector store creation rather than as an option in `findNearest()`. If developers were allowed to pass a different task type during a search, the query embedding would be mathematically incompatible with the stored document embeddings. To perform an accurate similarity search, the API would be forced to completely re-embed the entire vector store on the fly, which is computationally prohibitive.

### Chunking Strategy

To efficiently index long documents, the API automatically breaks the text into smaller segments called "chunks." This process is governed by the `chunkingStrategy`:
- `maxChunkSizeTokens`: Defines the upper limit for the size of each chunk.
- `chunkOverlapTokens`: Specifies the number of tokens that overlap between adjacent chunks. Overlap helps preserve context that might otherwise be lost if a concept is split precisely at a chunk boundary.

Developers can query `await VectorStore.params()` to discover the maximum supported chunk size and overlap limits for the underlying model.

### Privacy and Retention for Raw Text

A major consideration for the Vector Store API is the privacy and security
implication of persisting raw text. We propose two primary mechanisms to give
developers control over data retention and privacy:

#### The `storeContent` Configuration

Developers can explicitly opt out of storing raw text by setting a storeContent:
false configuration during vector store creation. In this mode, the browser
generates the embedding and immediately discards the raw text, persisting only
the embeddings and the developer-provided identifier (`id`).

**Caveat regarding Chunking and Identifiers**

When developers choose to discard the raw text, they rely entirely on the
returned `id` to map the search result back to their raw text. If the
developer inputs a massive document under a single `id`, the browser's
automatic chunking will split it into multiple embeddings that all map back to
that same `id`. During a search, the developer will know which document
matched, but they will not know which specific chunk or paragraph within the
document matched. To mitigate this 'lost context' problem, the search API can
return the chunk's original substring range (e.g., character start and end
offsets) alongside the search results. So developers can dynamically extract the
exact matching paragraph.

To maintain precise retrieval when using storeContent: false, developers can
keep their input strings short. By passing short, pre-chunked strings with
unique identifiers, developers ensure a strict 1:1 mapping between the search
result's identifier and the exact context matched.

#### Automatic Expiration (Time-To-Live)

To prevent abandoned, sensitive data from sitting on a user's device
indefinitely, the Vector Store API should implement a default expiration policy
(Time-To-Live, or TTL). 

This design can mirror industry standards, such as OpenAI's Vector Store API,
which supports an expires_after policy anchored to a last_active_at
timestamp. By using a sliding-window expiration, the browser automatically
resets the countdown every time the vector store is queried or updated. If the
web application is abandoned and the store remains inactive for a specified
number of days, the browser will automatically delete the vector store and its
contents, ensuring compliance with data retention best practices and freeing up
the user's disk space.

### Model Updates and Vector Migration

When a browser updates its embedding model, it can manage the migration of the
user's local Vector Store using the following mechanisms, depending on the
developer's configuration:

**Automatic Re-generation (`storeContent: true`)**

If the developer opted to store the raw text alongside the vectors (i.e.,
`storeContent: true`), the browser could re-generate embeddings with the new
embedding model.

**Handling Discarded Text (`storeContent: false`)**

If a developer used `storeContent: false`, the raw text is no longer available
to be re-embedded. To handle this scenario, there are two potential paths:

1.  **Forward Emulation Models:** To mitigate this without forcing a complete
    loss of the user's local database, we would love to explore with the
    community if and how Forward Emulation Models might help. An emulation model
    is a lightweight neural network (e.g., a Feed-Forward Neural Network)
    trained to translate vectors from the legacy embedding space directly into
    the new model's embedding space.
2.  **Automatic Deletion and Regeneration Events:** If emulation is not
    available, the vector store would be automatically deleted when the model is
    updated so the developer can detect that they need to regenerate it. Because
    relying on a missing database is not ideal, it's possible that a more
    explicit event would be useful for this case, proactively notifying the
    application that the store is about to be cleared and needs to be rebuilt.

**Ephemeral Stores**

If a developer creates a store for a temporary use case (such as real-time
moderation for a single chat session), they can configure the store with
ephemeral: true. This provides a strong signal to the browser that the vector
index has no value beyond the current session. The browser can aggressively
garbage collect it immediately upon closing the tab. During a model update, the
browser will entirely skip migrating or emulating these stores, intentionally
dropping them to conserve CPU cycles and battery life.

## Alternatives Considered

### API Architecture: Userland, Built-in Embeddings API, Built-in Vector Store API, or a Combination?

A major open question in the design of this API is which architectural layers
the web platform should natively provide versus what should be left to
developers. The three foundational approaches are: relying purely on userland
(WebAssembly/WebGPU), exposing a low-level raw Embedding API (e.g., [Semantic
Embedder API](https://github.com/explainers-by-googlers/semantic-embedder-api)),
or providing a fully managed high-level Vector Store API.

Industry platforms like OpenAI and Google Cloud often provide multi-tier
systems, offering both managed vector stores and raw embedding endpoints.
However, the Web Platform has unique constraints regarding cross-browser
interoperability and resource management. 

We are actively seeking input from other browser vendors, the Web Machine
Learning Community Group (WebML CG), and the broader web development community.
We are evaluating these core paths, as well as the potential of combining them
to serve different developer needs (e.g., offering both a built-in Vector Store
API and an Embedding API). We are exploring the following architectural options:

#### Option 1: Pure Userland (WebAssembly/WebGPU)

In this approach, the browser provides no built-in Embedding API or Vector Store
API. Developers must download their own model weights to the client, execute
them using WebAssembly or WebGPU, and build a custom vector database using a
pure JS library. 

**Target Developers:** Developers who require absolute control over the
embedding model to ensure strict client-server parity, or those who want to use
specific models not provided by the browser.

**Pros:**

1.  **Full Control Over the Model**: Developers have complete control over the
    model selection, which guarantees they can run the exact same logic on the
    server.
2.  **Leverages Existing Ecosystems:** Developers can rely on existing
    third-party libraries (e.g., LangChain, Qdrant, or transformers.js). This
    allows them to build advanced architectures immediately.
3.  **Enables Hybrid Local/Cloud Architecture**: Because developers can run the
    same model on both the server and the device, developers can compute a query
    embedding on-device and send it to an external cloud vector database (like
    BigQuery) to search massive remote datasets. This reduces cloud API costs
    and prevents raw user queries from being sent to third-party embedding APIs.
4.  **Enables ML Primitives Beyond Search:** Raw embeddings allow developers to
    build advanced features like clustering, anomaly detection, and complex
    multi-stage retrieval pipelines (e.g., re-ranking). It also empowers
    developers to implement their own retrieval and ranking system, chunking
    algorithm, and may even combine with lexical search to do hybrid search.
5.  **Debuggability and Control:** Developers can inspect the embeddings, debug
    distance metrics, and apply custom thresholding, giving them the necessary
    levers to fix issues if relevance quality drops.

**Cons:**

1.  **Storage / Memory Bloat and the "Shared Cache" Limitations for AI models:**
    Relying on userland means each site must download its own large model
    (\~40MB to 200MB), resulting in severe storage bloat and runtime memory
    thrashing when multiple parties run concurrent, taxing workloads using
    distinct models.
    -   While the [Cross-Origin Storage
        API](https://github.com/WICG/cross-origin-storage) could be seen as a
        mitigation, it fundamentally misaligns with developer incentives.
        Developers willing to absorb the cost of shipping a userland model do so
        specifically for competitive advantage, domain-specific fine-tuning, or
        strict control over their pipeline. The very nature of this
        customization means they are highly unlikely to coordinate and
        standardize on the exact same model weights as other sites (including
        competitors). Restricting developers to a curated list of models to
        prevent this bloat would defeat the primary advantage of userland
        (unlimited flexibility). Therefore, the statistical probability of a
        user experiencing a cache hit across different origins is near zero. 
    -   Conversely, developers who treat embeddings as a "commodity" capability
        (e.g., for basic semantic search or standard RAG) have no desire to
        manage, coordinate, or force their users to download large ML payloads.
        This large segment of developers is better served by a built-in API with
        a browser-managed model rather than a theoretical shared cache.
    -   Furthermore, relying on a shared cache creates an "innovation drag" and
        a "build up of storage pressure," as having the ecosystem upgrade to
        newer models breaks the cache and deduping benefits unless all
        coordinated sites somehow agree to upgrade simultaneously.
    -   Ultimately, the userland + shared cache approach for common AI
        capabilities normalizes popular third-parties (e.g., ads, analytics)
        downloading their own massive ML payloads, degrading the web's baseline
        performance with resource exhaustion and trashing, rather than solving
        the underlying resource problem.
2.  **Quantization Trade-offs:** While developers can use extreme quantization
    to compress models down to single-digit megabytes (e.g., 5-7MB), these
    primitives inherently sacrifice multilingual support, impose strict context
    bounds (e.g., 128 tokens), and reduce retrieval accuracy.
3.  **Restricted Background Migration & Global Orchestration:** If a model needs
    to be updated, the heavy re-embedding process is pushed into decentralized
    user-space. Because of site isolation, independent web apps cannot
    coordinate their resource usage, which risks simultaneous CPU spikes across
    origins.

#### Option 2: Built-in Embedding API Only

In this approach, the browser only provides an Embedding API that takes text and
returns raw embeddings. Developers would then be responsible for importing a JS
or WASM library (e.g., VectorDB.js, LangChain) to manage indexing and local
storage.

**Target Developers:** Developers who still need raw embeddings for custom ML
architectures, but want to offload the heavy model delivery, storage and
execution to the browser to improve performance and save user bandwidth. They
are willing to trade the "unlimited flexibility" of bringing their own model for
the sustainability of a browser-provided model, even if it means they have to
manage model versioning complexities.

**Pros:**

1.  **Retains Raw Embedding Capabilities Without the Bloat:** Like the userland
    approach, this API provides raw embeddings which means developers can still
    build hybrid local/cloud architectures, perform clustering, and retain full
    debuggability. However, it achieves this while eliminating the storage and
    memory bloat of shipping custom models.

**Cons:**

1.  **Interoperability Management:** While direct access to raw embeddings is
    strictly required to enable hybrid local/cloud architectures, it requires
    developers to explicitly handle model versioning and cross-browser
    fragmentation. Because browsers may use different models, the API must
    expose a model or embedding space identifier (e.g.,
    `embedding-gemma-300m`). Developers building server-side syncs must manage
    this configuration overhead, which introduces complexity compared to the
    fully encapsulated built-in Vector Store approach.
      - **Mitigation via Web-Native Content Negotiation:** To manage this
        fragmentation ergonomically, developers can leverage standard HTTP
        header negotiation (conceptually similar to Accept-Encoding). The client
        can transmit its raw embedding alongside its space identifier. If the
        server maintains a matching cloud vector database, it routes the query
        directly. If the server does not support that specific model space—or if
        the developer chooses not to maintain multiple vendor-specific
        indexes—the system can gracefully fall back to the client sending raw
        text, allowing the server to calculate the embedding using its preferred
        backend model.
      - Mitigation via Ecosystem Tooling and Translation: Similar to how a
        built-in Vector Store API can manage model upgrades locally, the broader
        open-source ecosystem (e.g., LangChain, popular vector databases) can
        abstract this interoperability burden on the server. By openly
        publishing the browsers' baseline model weights and collaborating on
        translation matrices, community libraries can offer middleware that
        automatically translates or routes browser-specific embeddings into
        common backend spaces. This provides developers the flexibility of raw
        embeddings with the convenience of a managed solution.

#### Option 3: Built-in Vector Store API Only

In this approach, developers can only insert text into the vector store API and
execute semantic searches against it. They never see the underlying numerical
embeddings.

**Target Developers:** Developers who require a fully managed, "plug-and-play"
semantic search solution that hides raw numerical vectors and automatically
handles data ingestion, chunking, indexing, on-device storage and model
migration.

**Pros:**

1.  **Structurally Enforced Interoperability:** While a low-level Embedding API
    allows for cross-browser interoperability if developers restrict themselves
    strictly to local execution, encapsulating the vectors enforces it by
    design. By encapsulating the vectors and preventing their export to external
    cloud vector databases, a vectorStore.findNearest() call guarantees
    functional interoperability for local-only retrieval across all browsers.
2.  **Automatic Model Migration:** The browser can silently migrate the hidden
    embeddings if the underlying model updates by directly re-generating the
    embeddings from the stored raw text. (Note: If the developer configures the
    store to discard the raw text for privacy reasons, silent migration becomes
    dependent on Forward Emulation Models; without them, the store must be
    cleared and rebuilt).
3.  **Platform-Managed Resource Stewardship & Protecting the Commons:** Because
    the vector store is a first-class platform API, the browser can actively
    manage the data lifecycle in the user's best interest. By baking lifecycle
    signals directly into the API design (e.g., explicit max-age constructs,
    eviction priorities, or session-bound ephemeral stores), the browser gathers
    the exact telemetry it needs to manage shared device resources intelligently
    when necessary.
    -   Unlike userland storage solutions, where developers rarely implement
        complex garbage collection, this native telemetry allows for nuanced,
        targeted optimization. The browser can transparently compress
        seldom-used embeddings or intelligently prune stale vectors without
        breaking the core application—and critically, it can do this in the
        background without waiting for the user to revisit the site to trigger
        elusive cleanup scripts.
    -   This solves the "tragedy of the commons" by ensuring high performance
        for the developer without permanently bloating the user's device, and
        avoids the browser having to resort to heavy-handed, origin-wide data
        evictions that unexpectedly break the user's web apps.
4.  **Future-Proofing the Web Platform:** The field of AI is evolving at an
    unprecedented pace. Having websites hardcode manual vector math, specific
    embedding dimensions, or today's tensor operations directly into website
    bundles (userland) risks locking web applications into transient technology.
    By exposing a high-level, declarative API (e.g., a VectorStore that handles
    the semantic matching abstractly), the browser shields the developer from
    the underlying implementation. This abstraction allows browser vendors to
    seamlessly upgrade models, swap hardware execution backends (e.g., moving
    workloads from GPU to dedicated NPUs), or even shift to entirely new
    semantic representation paradigms in the future—all without breaking a
    single existing web application, and without requiring a massive ecosystem
    to constantly update their dependencies.

**Cons:** 

1.  **Unusable for Hybrid Cloud Architectures:** Hiding the raw embeddings
    breaks interoperability with external systems. Developers who already
    maintain cloud vector databases will find this API entirely unusable for
    their architectures, completely blocking client-server hybrid retrieval use
    cases.
2.  **Blocks Advanced ML Use Cases:** Restricts developers to simple retrieval,
    blocking advanced use cases that require raw embeddings.
      - Clustering, e.g., group similar documents or topic discovery.
      - Complex multi-stage retrieval pipelines
          - Combined with re-ranking to achieve two-stage retrieval.
          - Combined with lexical search to achieve hybrid search.
          - Custom chunking algorithms.
      - Anomaly and outlier detection
      - Recommendation systems that combine embedding with other features.
3.  **Forces Client-Side IP Exposure:** Developers will be forced to download
    and expose their proprietary vector dbs to the client environment, which
    might be a non-starter for some.
4.  **Network and Storage Bloat:** Feature across origins will be forced to
    repeatedly download multi-megabyte vector dbs given that the vector
    operations need to happen on-device.
5.  **Opaque Debug Signals:** Because the underlying vectors are encapsulated,
    web applications cannot directly inspect the high-dimensional float arrays
    in production. This can make it difficult to troubleshoot edge-case semantic
    drift, or hamper fine-tuning distance thresholds if search relevance is poor
    in a specific browser.
      - **Potential Mitigation:** While raw vector access might be restricted in
        production to prevent premature dependency on browser-specific embedding
        models, the browser could expose deep diagnostic tools exclusively for
        developers (e.g. a DevTools feature) and automated testing environments.
6.  **Practical Implications for Quality and Configuration:**
      - **Variable Search Quality**: While the API shape remains interoperable,
        the quality of the search results will not be identical across browsers
        as different models may return more or less relevant results. 
          - **Potential mitigation:** Rather than leaving retrieval quality an
            opaque variable, we are interested in working with the community and
            other browser vendors to establish a standardized baseline quality
            threshold using widely accepted industry metrics, such as the
            Massive Text Embedding Benchmark (MTEB). For example, browser
            vendors could agree that any model powering the built-in API must
            achieve a minimum MTEB score (e.g., a score of 60 or higher)
      - **Chunking and Token Limits:** Some embedding models have different
        maximum input token limits (e.g., 512 vs. 2048 tokens). To maintain
        cross-browser compatibility, developers would either have to rely
        entirely on a browser-determined `auto` chunking strategy, or the API
        would need to expose capability ranges (e.g., maximum supported chunk
        size bounds, which will likely be exposed via `VectorStore.params()`),
        so developers can dynamically configure their chunking logic per
        browser. Some developers also prefer custom, context-aware chunking
        algorithms for their specific use cases.

#### Option 4: Two-Tier APIs (Vector Store + Embedding API)

This approach provides the managed VectorStore API for convenience, while also
exposing a separate Embedding API to generate raw vectors for advanced ML
workloads.

**Target Developers:** The entire spectrum of developers, from developers
seeking a fully managed, "plug-and-play" semantic search solution, to developers
requiring raw vector access for hybrid cloud architectures and custom ML
pipelines.

**Pros:**

1.  **Widest Applicability:** Satisfies the broadest range of use cases. It
    provides a frictionless "plug-and-play" solution (Vector Store API), while
    simultaneously preserving the critical hybrid local/cloud architectures
    required by advanced integrations (Embedding API).

**Cons:**

1.  **Resource Allocation:** While this architecture maximizes utility, it
    requires a larger upfront investment from the community and browser vendors
    to implement and maintain both architectural layers concurrently, rather
    than forcing a single, compromised solution onto the ecosystem.

## 

### WebAssembly/WebGPU (DIY)

See discussion above.

### Browser-provided Embedding API

See discussion above.

### 

### Cloud APIs

Provides the highest quality models and additional computing resources, at the
cost of user privacy and financial expense for the developer. Other downsides:

  - **Latency:** Cloud APIs require network round-trips, which ties performance
    requirements to network characteristics.
  - **Lack of offline capability:** Cloud APIs cannot support scenarios
    where offline or weak network conditions are expected.

### On Commoditization of On-Device AI models and Standardization

A common critique of built-in AI APIs powered by browser-managed models is the
risk of fragmentation: what if Chrome ships Model A and Safari ships Model B?

While fragmentation is a risk in the nascent stages of any new web capability,
historical precedent suggests a different endgame. Just as the web platform—and
more importantly, developers—eventually coalesces around standard image formats,
video codecs, and cryptographic algorithms, base-level semantic embeddings are
rapidly becoming a commodity.

Ultimately, having different browsers ship slightly different base embedding
models provides no competitive advantage to the browser, and only inflicts pain
on web developers. The long-term vision of these APIs—and any web platform API,
for that matter—is not to create browser differentiation, but to drive
innovation and standardize on a common set of fundamental models that all
vendors can agree to ship for the benefit of the web and its users.

#### The Standardization Timeline as an Incubation Period

It is a reality of the Web Platform that cross-vendor consensus and adoption
take time. However, in the context of AI, this timeline works to our advantage.
The many months and years required to debate, refine, and cross-pollinate AI
APIs across different browser engines provide the exact incubation period the
broader AI industry needs to settle on heavily optimized, universally accepted
baseline models (the "lingua franca of embeddings").

By discussing and co-designing a high-level API with the community now, we are
not prematurely fixing the web into today's fragmented AI model landscape;
rather, we are building the structural plumbing required for that inevitable,
commoditized future, and all the upsides and ergonomics it entails.

## Internationalization Considerations

Because different embedding models have different levels of language support, the API needs to account for content in various languages. To address this, we are considering exposing supported languages via `VectorStore.params()` (similar to other built-in AI APIs like the Prompt API) and requiring developers to provide a `language` parameter within the `create()` options. This design allows the API to fail early with an error if the user's device lacks a suitable embedding model for the requested language, ensuring developers can gracefully fall back to alternative architectures.

## Security Considerations

  - **Permissions Policy:** We are evaluating whether access to the API should
    be gated by a strict permissions policy. Because chunking, embedding, and
    indexing are heavy computational tasks, developers may prefer to use this
    API within Web Workers. Given the known spec friction of applying
    permissions policies to Workers, we will align this requirement with the
    broader Built-in AI ecosystem's general solution for Worker security
    contexts.
  - **Sandbox Isolation:** Data processing and model execution should occur in a
    sandboxed environment to mitigate risks of malicious inputs.

## Privacy Considerations

- **Data Locality:** All text and generated vectors remain entirely on the
user's device. 

- **Cross-Site Tracking:** Vector stores must be partitioned by the same
storage partitioning concepts used by IndexedDB, to prevent cross-site tracking
or data leakage.

- **Storage Limits:** Because vector indexes can grow large, the API must
adhere to the standard web storage quota rule. If an origin exceeds its allowed
storage during an active session, operations like insert() will throw a standard
QuotaExceededError DOMException.

- **Privacy and Retention for Raw Text**: See discussion above.

## Future Explorations

As this API evolves beyond the initial retrieval MVP, the community and browser
vendors will need to explore the following questions:

1.  **Advanced ML Capabilities in the Managed API:** While the V1 Vector Store
    is strictly scoped to semantic retrieval, should future versions natively
    expose advanced ML features (such as clustering or anomaly detection), or
    should those remain strictly in userland built on top of the low-level
    Embedding API to prevent API scope creep?
2.  **UI Chunk Highlighting and Substring Metadata:** How should the API expose
    exact chunk substring ranges and identifiers to allow frontend developers to
    build advanced UI features (like highlighting the exact matching sentence
    within a larger rendered document)?
3.  **Obfuscated Vector Spaces:** Is there a viable path to offer an
    "obfuscated" low-level vector space that prevents cross-origin IP leakage
    while maintaining cross-browser interoperability, or does obfuscation
    fundamentally break the enterprise use-case of hybrid local/cloud vector
    search?
4.  **Multi-Modal Embedding Support:** While this initial proposal is strictly
    scoped to text to stabilize the core database and chunking mechanics,
    multi-modal search (e.g., querying local images or audio using text) is a
    critical target use case. Future iterations of this API will explore how to
    efficiently ingest and process binary data structures (such as Blob,
    ImageBitmap, or AudioBuffer) and handle cross-modal embedding spaces within
    the managed store without degrading client performance.

## Polyfill / Try It Out

To allow developers to experiment with the proposed built-in Vector Store API
syntax, we have published a [syntactic polyfill for the Vector Store
API](https://github.com/KenjiBaheux/Vector-Store-Polyfill). This polyfill is
built in userland on top of the raw [Semantic Embedder
API](https://github.com/explainers-by-googlers/semantic-embedder-api).

**Important Disclaimer:** This polyfill is strictly a short-term companion
project designed to give developers a working preview of the API surface. It is
not a production library. Because it operates in userland, it fundamentally
lacks the system-level capabilities that make a native built-in Vector Store API
valuable, including: opaque storage, platform-managed resource stewardship,
platform-orchestrated model migration, etc. See userland discussion for more
details.

## Prior Art

### OpenAI Vector Store API

```javascript
import OpenAI from "openai";
const openai = new OpenAI();
// Create
const vectorStore = await openai.vectorStores.create({
  name: "Support FAQ"
  // optional chunking strategy
});
// List
const vectorStores = await openai.vectorStores.list();
// Retrieve
const vectorStore = await openai.vectorStores.retrieve(
  "vs_abc123"
);
// Insert file
const myVectorStoreFile = await openai.vectorStores.files.create(
  "vs_abc123",
  {
    file_id: "file-abc123"
  }
);
// Search
const result = await openai.vectorStores.search('vs_abc123', {
  query: 'string',
});
// Delete
const deletedVectorStore = await openai.vectorStores.delete(
  "vs_abc123"
);
```

### LangChain

```javascript
import { OpenAIEmbeddings } from "@langchain/openai";
import { MemoryVectorStore } from "@langchain/classic/vectorstores/memory";
// Create
const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",
});
const vectorStore = new MemoryVectorStore(embeddings);
// Insert data
import { Document } from "@langchain/core/documents";
const document = new Document({
  pageContent: "Hello world",
});
await vectorStore.addDocuments([document]);
// Search
const results = await vectorStore.similaritySearch("Hello world", 10);
// Delete
await vectorStore.delete({
  filter: {
    pageContent: "Hello world",
  },
});
```

### Weaviate

```javascript
import weaviate, { WeaviateClient, vectors } from 'weaviate-client';
const client: WeaviateClient = await weaviate.connectToLocal();
// Create
await client.collections.create({
  name: 'Question',
  vectorizers: vectors.text2VecOllama({
    apiEndpoint: 'http://ollama:11434',
    model: 'nomic-embed-text',
  }),
});
// List
const allCollections = await client.collections.listAll();
// Retrieve
const questions = client.collections.use('Question');
// Insert data
const data = await getJsonData(); //helper function
const result = await questions.data.insertMany(data);
// Search
const result = await questions.query.nearText('biology', {
  limit: 2,
});
// Delete
await client.collections.delete('Question');
```

### BigQuery Vector Database

```sql
# Create
CREATE TABLE mydataset.products (
  name STRING,
  description STRING,
  description_embedding STRUCT<result ARRAY<FLOAT64>, status STRING>
    GENERATED ALWAYS AS (
      AI.EMBED(description, model => 'embeddinggemma-300m')
    ) STORED OPTIONS( asynchronous = TRUE )
);
# Insert data
INSERT INTO mydataset.products (name, description) VALUES
  ("Lounger chair", "A comfortable chair for relaxing in."),
  ("Super slingers", "An exciting board game for the whole family."),
  ("Encyclopedia set", "A collection of informational books.");
# Search
SELECT *
FROM
  VECTOR_SEARCH(
    TABLE mydataset.products,
    'A really fun toy',
    (SELECT image_ref FROM mydataset.images_query)
  );
# List, Retrieve, Delete are regular SQL operations for tables.
```

