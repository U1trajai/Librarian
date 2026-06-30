# The Librarian — Work Breakdown Structure

## Goal

Build an MCP-accessible retrieval system that indexes our codebase first, then expands to Confluence docs and GitHub issues/PRs, surfacing the most relevant content to Claude (or any MCP client) on demand — without requiring full context to be loaded upfront. This document scopes the **MVP**: GitHub code only, end to end.

## Locked-in stack

| Layer | Choice | Why |
|---|---|---|
| Vector DB | Qdrant (self-hosted, Docker) | Strong filtering, native hybrid (dense+sparse) search, simple to self-host |
| Embeddings | `BAAI/bge-m3` | Single model produces both dense and sparse vectors — no need to juggle a separate keyword index |
| Reranker | `BAAI/bge-reranker-v2-m3` | Pairs naturally with bge-m3; strong open-source cross-encoder |
| Model serving | Infinity server (Docker) | One server handles both embedding and reranking — less infra to babysit solo |
| Ingestion state | SQLite | Lightweight tracking of sync timestamps / seen IDs per source, separate from vector data |
| Interface | MCP server (Python SDK) | Works in Claude Code and Claude.ai without custom plugin work |
| Scope | Solo/local first | Defer multi-user auth and remote hosting until the core pipeline is proven |

**Deferred for now (explicitly out of scope for v1):** team-wide remote hosting, per-user auth, multi-tenant access control, non-English content, multilingual reranking. Revisit after MVP validation.

---

## Phasing strategy

Build the thinnest possible end-to-end slice first — **GitHub code only, no Confluence, no reranking** — to validate that the architecture works before investing in every source type. This avoids a common failure mode: building all of ingestion, then all of chunking, then discovering at the very end that retrieval quality is poor and not knowing which layer caused it.

This document covers that MVP slice in full detail (Phases 0–6 below). Everything beyond it — Confluence, GitHub issues/PRs, reranking, formal evaluation, and team rollout — is intentionally pushed to a single **"Beyond MVP"** section at the end, so the plan stays focused on what's actually being built first.

```
Phase 0 → Phase 1 → Phase 2 (GitHub code ingestion) →
Phase 3 (chunking, code only) → Phase 4 (embed/index) →
Phase 5 (retrieval) → Phase 6 (MCP server)
= MVP, usable end to end
```

---

## Phase 0 — Environment setup

**Goal:** Get the core infra running locally before writing any pipeline code.

- Stand up Qdrant via Docker Compose
- Stand up Infinity server via Docker, load `bge-m3` and `bge-reranker-v2-m3`
- Verify both with a manual curl/Python smoke test (embed a string, search an empty collection, rerank two dummy strings)
- Set up project repo structure: `ingestion/`, `chunking/`, `indexing/`, `retrieval/`, `mcp_server/`, `state.db`

**Definition of done:** You can embed a test string and store/retrieve it from Qdrant, and rerank two dummy candidates, all running locally.

**Estimate:** 0.5–1 day

---

## Phase 1 — Sync state tracking

**Goal:** Build the SQLite layer that every ingestion source will use, before building any actual ingestion.

- Schema: `sources` table (source_type, source_id, last_synced, content_hash, status)
- Functions: `mark_synced(source_id, hash)`, `get_stale_or_new(source_type)`, `get_deleted(source_type, current_ids)`
- This is what lets every later ingestion job be incremental instead of full-reindex-every-time

**Definition of done:** Unit tests pass for detecting new, changed, and deleted items given two mock ID/hash sets.

**Estimate:** 0.5 day

---

## Phase 2 — GitHub code ingestion

**Goal:** Pull code from your repos into local working copies, ready for chunking.

- Use `git clone`/`git pull` (not the API) for bulk file content
- Walk the tree, filter: skip `node_modules`, `vendor`, lockfiles, generated/build dirs, binary files — define an explicit allowlist of extensions rather than a denylist (less likely to let junk through)
- Compute a content hash per file; compare against Phase 1 state to find new/changed/deleted files
- Output: a list of (file_path, repo, content, content_hash) tuples ready for chunking

**Definition of done:** Running the script twice in a row with no upstream changes results in zero files being marked for reprocessing.

**Estimate:** 1–1.5 days

---

## Phase 3 — Chunking pipeline (code)

**Goal:** Turn raw ingested code into retrieval-ready chunks.

- Use `tree-sitter` with the relevant language grammar(s) for your codebase
- Chunk by function/method/class definition
- Each chunk includes: the code body, its docstring/leading comment, its signature, and the enclosing class name if applicable
- For functions exceeding a size threshold (e.g. ~800 tokens), fall back to splitting at nested block boundaries rather than mid-line

**Definition of done:** Given a sample file, chunk boundaries align with logical units (visually inspect 20–30 sample chunks) and no chunk silently truncates a function.

**Estimate:** 2–2.5 days

---

## Phase 4 — Embedding & indexing

**Goal:** Turn chunks into vectors and load them into Qdrant with metadata.

- Define the Qdrant collection schema: dense vector (bge-m3 output), sparse vector (bge-m3 sparse output), and payload fields — `source_type`, `source_id`, `title`/breadcrumb, `url`, `last_updated`, `repo_or_space`, `chunk_type`
- Batch-embed chunks via the Infinity server, upsert into Qdrant by a stable chunk ID (e.g. hash of source_id + chunk index) so re-ingestion overwrites rather than duplicates
- On Phase 1/2 detecting a deletion, delete the corresponding chunk IDs from Qdrant in the same pass

**Definition of done:** Indexing a small test repo populates Qdrant with the expected chunk count, payload fields are all present and correctly typed, and re-running indexing after a single file change updates only that file's chunks.

**Estimate:** 1–1.5 days

---

## Phase 5 — Retrieval (hybrid search)

**Goal:** Query-time search combining dense + sparse vectors with optional metadata filters.

- Implement hybrid search via Qdrant's native query API (dense + sparse fusion)
- Support optional filters: `source_type`, `repo_or_space`, `last_updated` range
- Return top-N (e.g. 20–30) candidates with scores and payload

**Definition of done:** A handful of manually-written test queries against your indexed test repo return the expected file/function in the top 5 results, unfiltered.

**Estimate:** 1 day

---

## Phase 6 — MCP server (MVP interface)

**Goal:** Expose retrieval as a tool any MCP client can call.

- `search_knowledge_base(query, source_type?, repo_or_space?, max_results?)` — calls Phase 5 retrieval, returns chunks with title/url/last_updated
- `get_document(source_id)` — returns the full original file/page content for cases where a chunk isn't enough
- Write tool descriptions deliberately — state explicitly what kinds of questions the tool is good for, since the model decides when to call it based on the description text
- Run as a stdio MCP server initially; wire into Claude Code's MCP config for local testing

**Definition of done:** From within Claude Code, you can ask a question whose answer lives in your indexed repo, and Claude calls the tool, retrieves the right chunk, and answers correctly with a source link.

**🎯 This is the MVP milestone — a working end-to-end slice on GitHub code only.**

**Estimate:** 1.5–2 days

---

## Beyond MVP (not detailed yet — for context only)

Once the MVP milestone above is demoed and validated, the natural next additions, roughly in priority order:

1. **GitHub issues/PRs ingestion** — via GraphQL API, same incremental-sync pattern as code (track `updatedAt` against Phase 1 state)
2. **Confluence ingestion** — REST API v2, storage-format pages converted to Markdown, chunked by heading hierarchy with a title/heading breadcrumb prepended to each chunk
3. **Reranking** — add the Infinity reranker (`bge-reranker-v2-m3`) on top of Phase 5's hybrid search results to improve precision
4. **Formal evaluation** — a hand-written eval set (20–30 questions with known correct sources) and a recall@K script, so future changes can be measured rather than eyeballed
5. **Team rollout hardening** — move the MCP server from stdio to HTTP/SSE, centralize Confluence/GitHub credentials as a service account, add webhook-based sync, add access scoping if needed

Each of these deserves its own short scoping pass (goal, steps, definition of done, estimate) once it's actually next up — intentionally left unscoped here to keep this document focused on the MVP.

---

## Rough total effort (MVP: Phases 0–6)

| Phase | Estimate |
|---|---|
| 0 — Environment setup | 0.5–1 day |
| 1 — Sync state tracking | 0.5 day |
| 2 — GitHub code ingestion | 1–1.5 days |
| 3 — Chunking (code) | 2–2.5 days |
| 4 — Embedding & indexing | 1–1.5 days |
| 5 — Retrieval | 1 day |
| 6 — MCP server | 1.5–2 days |
| **MVP total** | **~8–10 working days** |

These are focused-work estimates, not elapsed calendar time — pad for context-switching if this isn't your only responsibility.

## What to show your team at the MVP milestone

Once Phase 6 is done, you'll be able to demo: ask Claude Code a real question about your codebase, watch it call the search tool, and get a correctly-sourced answer with a link back to the file. That's the concrete proof point worth presenting before committing to the larger build-out listed under "Beyond MVP" above.
