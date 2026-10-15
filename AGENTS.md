# AGENTS.md

This repository is the **public, documentation-only** companion to the Micro-RAG Knowledge Engine by Ruel T. Abion (https://ruelabion.com). The implementation lives in a private repository and is shared on request.

## Rules

- Documentation only. Do not add source code, scripts, configuration, sample data or tests. Short pseudo-code in `docs/WALKTHROUGH.md` is acceptable.
- Never add AWS account IDs, resource IDs, ARNs, credentials, private hostnames, client names or internal project names. Scan before every commit.
- Keep claims honest and in step with the browser demo (https://rag.ruelabion.com) and the private repository: do not call the keyword retriever semantic, do not claim tests against the live Claude API beyond stubbed requests, and do not add benchmarks or accuracy figures without evidence.
- Never imply the code is public. The code-request path is https://ruelabion.com/#request-code.
- Do not describe the repository as published code, and do not add licence text that grants rights to software not included here.
- Update `CHANGELOG.md` with every change.

## Keeping in step

When behaviour or limits of the project change, the private repository updates `app/lib/faq.ts`, `demo/public/llms.txt` and `docs/HOW-IT-WORKS.md`. Update `README.md`, `docs/HOW-IT-WORKS.md` and `docs/LIMITATIONS.md` here to match.

## Git

Focused imperative commits. Co-authored AI commits should carry the attribution line the session provides.
