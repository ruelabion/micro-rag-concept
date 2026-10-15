# How the browser demo works without a server, Docker or an LLM

The public demo at [rag.ruelabion.com](https://rag.ruelabion.com) runs the whole retrieval pipeline inside the visitor's browser. This page explains how, stage by stage. It describes the pipeline the full application and the demo share. The implementation is not in this repository.

## What is replaced, and by what

| A production RAG system normally uses | The browser demo uses |
|---|---|
| A vector database (for example pgvector in Docker) | An array held in the page's memory |
| An embedding model behind an API | A feature-hashing function of about 40 lines |
| An LLM to write the answer | A generator that quotes the best passages |
| A server to run the pipeline | The visitor's browser |

The page itself is static files on S3 and CloudFront. Once it has loaded, asking a question makes no network request.

Each substitute implements the same small interface as the real component (vector store, embedding provider, generator), which is why the full application can swap in pgvector, a real embedding model and the Claude API without changing the stage code.

## Ingestion

Ingestion runs when the page loads. The five sample Markdown files were embedded in the page at build time, so no server is needed to fetch them. The pipeline handles each document in turn:

1. **Skip unchanged documents.** A pure-JavaScript SHA-256 hashes the text and the store is asked whether it already holds that hash. If it does, the document is skipped with no further work. In the demo the store starts empty, so nothing is skipped; in the full application this makes re-ingestion free.
2. **Chunk** :
   - split on blank lines into paragraphs, tracking Markdown headings;
   - merge paragraphs into chunks of up to about 1,000 characters;
   - split a paragraph only if it is longer than that, and then on sentence boundaries;
   - prefix each chunk with its section heading so a passage keeps its context;
   - repeat 120 characters of the previous chunk within a section so a fact is not cut at a boundary.

   The five sample documents become 23 chunks.
3. **Embed** each chunk into a vector of 384 numbers:
   - lowercase the text, keep words and numbers, drop stopwords, strip common suffixes lightly (so "retained" lands near "retention");
   - count each word, plus each pair of adjacent words at half weight;
   - hash each word with FNV-1a; the hash chooses one of the 384 slots and a sign, and the slot receives `1 + log(count)`;
   - scale the vector to length 1.

   Texts that share words fill the same slots, so their vectors point the same way.
4. **Store atomically.** `replaceDocument` saves the chunks with their vectors and replaces any earlier chunks of that document.

All of this takes a few milliseconds.

## Asking a question

Asking a question runs these steps:

1. Embeds the question with the same function.
2. Computes cosine similarity against every stored vector (a dot product, since the vectors are unit length) and takes the top 4.
3. Drops anything scoring below 0.2. If nothing is left, it **refuses and never calls the generator**.
4. Otherwise the extractive generator takes the first two sentences of the top two passages and tags them `[1]` and `[2]`. It cannot invent text because it only quotes, and for the same reason it cannot reason or combine passages.
5. The citation validator keeps only markers that point at passages actually supplied, and flags an answer with no valid citation as unverified.
6. The result is returned with a trace: scores, timings, the embedding model name and any dropped markers.

The demo links every citation to the full source document so a visitor can compare the answer with the original wording.

## What this does and does not show

- The vectors capture **shared words, not meaning**. "Paid time off" will not match "vacation", and an off-topic question that shares a few words with the sample handbook can still clear the threshold. That is why refusal checks in the evaluation script are informational unless `--strict` is used.
- The extractive generator can quote the wrong sentences from the right document.
- The demo shows the pipeline's **behaviour** (retrieve, cite, refuse) and its structure. It does not show language-model quality or semantic search.
- Validation proves a citation points at a supplied passage, not that the passage supports a claim.

For the design reasoning see [ARCHITECTURE.md](ARCHITECTURE.md); for honest limits see [LIMITATIONS.md](LIMITATIONS.md). To see the behaviour, try [rag.ruelabion.com](https://rag.ruelabion.com). The code is available on request through [ruelabion.com/#request-code](https://ruelabion.com/#request-code).
