# Walkthrough: how the pattern works

This is the idea in implementation order. It is a description of the pattern, not the project's code.

## 1. Split documents into passages

Cut each document into passages of a few sentences, with a small overlap so a fact isn't lost at a boundary. Keep the source name and the passage's position, because the citation will show them. Hash each passage so unchanged content can be skipped on re-ingestion.

## 2. Embed and store

Turn each passage into a vector with an embedding model and store it with its text and source. Record which embedding model produced the vectors, and refuse to mix models in one index.

## 3. Retrieve with a threshold

Embed the question the same way and take the top few passages by cosine similarity. **Drop any passage below a minimum score.** The threshold is the first line of defence against inventing an answer.

```text
passages = top_k(similarity(question, index), k)
passages = [p for p in passages if p.score >= MIN_SCORE]
if passages is empty: return REFUSAL        # no model call at all
```

## 4. Generate from the passages only

Number the passages `[1]..[N]` and tell the model to answer only from them, cite with those numbers, and say when they are insufficient. Wrap the passages so the model treats them as data, never as instructions.

## 5. Validate the citations

Parse the `[n]` markers in the output. Keep a marker only if `1 <= n <= N`. Remove invented ones and record them. If none remain, mark the answer as unverified instead of presenting it as grounded.

```text
valid   = {n for n in markers(answer) if 1 <= n <= N}
dropped = markers(answer) - valid
answer.ungrounded = (valid is empty)
```

## 6. Return a trace

Return the scores, the embedding model, timings and any dropped markers with every answer. When an answer is wrong, the trace shows whether retrieval or generation failed.

## 7. Evaluate it

Keep a small labelled question set: questions that should be answerable (with the expected source) and questions that should be refused. Gate changes on whether retrieval finds the expected source. Treat the refusal rate as informational until you use a semantic embedding model, because keyword scores for off-topic and on-topic questions overlap.

## Try the behaviour

Open [rag.ruelabion.com](https://rag.ruelabion.com), ask "How quickly must on-call acknowledge a Severity 1 incident?", and follow the citation to the full source. Then ask an unrelated question and watch the pipeline decline.
