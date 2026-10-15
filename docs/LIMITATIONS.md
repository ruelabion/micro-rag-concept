# What this does and doesn't prove

Honest limits of the project and its public demo.

- **Citation validation is structural.** It confirms a citation points at a passage that was supplied to the generator. It does not confirm the passage supports the claim. The demo links every citation to the full source so you can check.
- **The demo's retriever is lexical.** It matches overlapping words, not meaning. It cannot always separate an off-topic question from an in-scope one (for example, a question about an HR policy that shares words with the sample handbook may be answered). A semantic embedding model is the production answer, with the refusal threshold tuned on a labelled set.
- **The demo's generator is extractive.** It quotes passages and can pick the wrong sentences from the right document. It shows the pipeline's behaviour, not language-model quality.
- **The offline test embedder proves plumbing only.** Retrieval quality claims need a real embedding model and your own documents.
- **Live provider calls are lightly verified.** The vector store was exercised against a real pgvector database. The Claude request shape is tested against a stubbed HTTP layer; the live API path hasn't been load-tested.
- **Prompt injection is reduced, not eliminated.** Treating retrieved text as data helps; hostile documents still need review at ingestion.
- **Sample data is generic.** The bundled documents contain no client data or private information.
- **No security or compliance guarantees.** Treat it as a reference design and review it against your own requirements.

Questions about taking this pattern to production: [ruelabion.com/#assessment](https://ruelabion.com/#assessment).
