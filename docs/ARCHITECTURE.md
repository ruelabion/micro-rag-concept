# Architecture and design decisions

## Goals

1. Answers must be traceable to specific source passages.
2. When the sources don't contain the answer, say so instead of guessing.
3. Every stage must be observable and replaceable without touching the others.
4. The whole pipeline must be testable offline.

## Stages and boundaries

| Stage | Input | Output | Replaceable via |
|---|---|---|---|
| Ingestion | documents | passages with vectors, stored | embedding provider, vector store |
| Retrieval | question | scored passages above a threshold | embedding provider, vector store |
| Generation | question + passages | text with `[n]` markers | generator |
| Validation | text + passages | answer with only real citations | none; a pure function |

The core has no dependency on the web framework. The HTTP layer, the command-line scripts and the tests all call the same functions.

## Decisions

### Separate ingestion, retrieval and generation
Each stage has one job and a typed interface. Failures stay attributable (a bad answer is either a retrieval problem or a generation problem, and the trace says which), the model vendor can be swapped without re-indexing, and tests can drive each stage with a fake.

### Refuse before generating
If no passage reaches the minimum score, the generator is never called. This saves cost and removes the main source of confident but ungrounded answers: a model handed irrelevant context will often try to answer anyway. A test asserts the generator is not invoked on this path.

### Validate citations after generation
The model is asked to cite with `[n]`. The pipeline keeps only markers in `1..N`, where N is the number of passages actually supplied, drops the rest and records them in the trace. An answer with no valid citation is returned flagged as ungrounded. This checks that a citation *exists*, not that it *supports* the claim; that would need a separate verification step.

### Treat retrieved text as untrusted data
Passages are wrapped in delimiters and the system prompt states they are data, not instructions. Attribute values are stripped of characters that could break out of the delimiter. This reduces, but does not eliminate, prompt-injection risk from ingested documents.

### Idempotent, content-addressed ingestion
A document's identity comes from its source name, and its content hash is stored. Re-ingesting unchanged content makes no embedding calls. Changed content replaces all of that document's passages in one transaction, so a document is never half-updated.

### Pin the embedding model to the index
The store records the embedding model and dimensions on first use and refuses a different configuration. Mixing vectors from two models silently ruins retrieval; failing loudly is cheaper than debugging it.

### Provider boundaries are single files
The model and embedding providers use plain HTTP with timeouts, bounded retries on 429 and 5xx, and response validation. There is no vendor SDK to upgrade and the request shape is visible in one place.

### pgvector with an HNSW cosine index
Postgres keeps documents, passages and vectors in one transactional system, which is enough for the corpus sizes this pattern targets. HNSW provides approximate nearest-neighbour search without a training step.

### An offline embedder so tests need no network
A feature-hashing embedder over normalized words and bigrams makes tests deterministic and free. It is deliberately labelled lexical: it proves the plumbing and says nothing about semantic quality.

### Bounded cost and abuse
The full application caps request size and rate, keeps a daily ceiling on questions and ingestion, and does not trust forwarded-for headers unless told how many proxies to trust. The browser demo makes no model calls, so a flood can cost only bandwidth.

## Deliberately out of scope

| Missing | Why | For production |
|---|---|---|
| Hybrid search and reranking | Hides the core pattern | A keyword candidate set plus a reranker |
| Streaming responses | Citation validation needs the full text | Stream, then validate and append citations |
| Auth and tenancy | Depends on the host application | Per-tenant filters enforced server-side |
| Distributed rate limiting | Needs shared state | Enforce at the gateway or in a shared store |
| Document extraction (PDF, HTML) | Orthogonal to the pipeline | An extraction step before ingestion |
| Answer-support verification | Needs a second model call | An entailment or quote-check pass |
