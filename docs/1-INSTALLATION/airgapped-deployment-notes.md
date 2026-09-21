# Air-Gapped Deployment: Working Notes

Companion to the [Air-Gapped Deployment Runbook](airgapped-deployment.md). The runbook tells you what to do; this file records *why*, with references into the codebase, so the next person does not have to re-derive it.

The behaviour below was established by reading the code rather than by running a deployment. Line references drift — treat them as starting points, not guarantees.

---

## Target environment these notes assume

RHEL on both the staging VM and the offline host. vLLM already serves an OpenAI-compatible chat endpoint; a second vLLM instance serves embeddings. OCR via Docling is in scope, so a derived image is built. Podcasts and audio transcription are out of scope. Single user.

---

## Verified behaviour

### Only two model defaults are mandatory

`api/routers/models.py` defines `REQUIRED_DEFAULTS = {"default_chat_model", "default_embedding_model"}`. These can be reassigned but never cleared. Every other slot — transformation, tools, large context, TTS, STT — falls back to the chat model or is skipped when empty, and `auto-assign` deliberately leaves them alone (see #1097/#1098).

A minimal deployment therefore needs one language model and one embedding model, nothing more.

### SurrealDB is the vector store

There is no separate vector database. `source_embedding.embedding` is declared `array<float>` in migration 1; `note` and `source_insight` carry their own `embedding` fields.

Search is hybrid. `fn::text_search` uses a real BM25 full-text index (`idx_source_embed_chunk ... SEARCH ANALYZER my_analyzer BM25 HIGHLIGHTS`). `fn::vector_search` computes `vector::similarity::cosine`, filters on a minimum similarity, orders and limits.

**There is no approximate-nearest-neighbour index.** No MTREE or HNSW appears in any migration; the only indexes on embedding tables are the BM25 text index and a plain index on the `source` foreign key (migration 10). Vector search is a brute-force linear scan. Fine for hundreds of documents, linear in corpus size beyond that.

Migration 24 is the current definition of both search functions — it added notebook scoping and fixed the ordering. The version in migration 1 applied `LIMIT` before computing similarity; do not read migration 1 as current behaviour.

### The embedding model choice is effectively permanent

Nothing enforces a fixed dimension, so swapping embedding models is not blocked — the stored vectors simply stop being comparable to new queries. Changing it means re-embedding every source.

### tiktoken works offline out of the box

The Dockerfile pre-downloads the `o200k_base` encoding at build time into `/app/tiktoken-cache`, deliberately **outside** `/app/data` so a volume mount cannot shadow it (issue #264). `TIKTOKEN_CACHE_DIR` points at it.

`open_notebook/utils/token_utils.py` also catches `(ImportError, OSError)` and falls back to `len(text.split()) * 1.3`. That fallback is silent apart from a warning and shifts chunk boundaries and the 105k large-context threshold, so it matters for source installs where the cache is not seeded.

### One outbound call remains, with no kill switch

`api/routers/config.py` calls `get_version_from_github_async(...)`, which fetches `pyproject.toml` from `raw.githubusercontent.com`. It is triggered by `GET /api/config`, which sits in `PasswordAuthMiddleware`'s `excluded_paths` and is therefore unauthenticated, and which `ConnectionGuard` calls on essentially every app load.

Mitigations already in place: a 24-hour cache that also caches failures, a 10-second timeout, and a warning log rather than a crash. A grep for `DISABLE_VERSION|VERSION_CHECK|OFFLINE_MODE|AIRGAP` returns nothing — there is no way to switch it off.

Prefer a firewall rule that REJECTs over one that DROPs. A blackholed connection burns the full timeout while `ConnectionGuard` blocks rendering.

### Sub-path hosting is not possible without a rebuild

`frontend/next.config.ts` sets no `basePath` or `assetPrefix`, and no `BASE_PATH` variable exists anywhere in the frontend. FastAPI sets no `root_path`. In Next.js `basePath` is baked in at build time.

Three things break under `example.com/onb/`: static assets resolve to `/_next/...` from the root; `frontend/src/lib/config.ts` fetches `/config` and `${baseUrl}/api/config` with `baseUrl` defaulting to an empty string; and `frontend/src/lib/api/client.ts` does `window.location.href = '/login'` on a 401, a raw browser navigation that ignores `basePath` even when one is set.

Every reverse-proxy example in `docs/5-CONFIGURATION/reverse-proxy.md` is a `location /` block on a dedicated hostname. Use a hostname.

### A scanned PDF fails loudly, not silently

Without Docling, `auto` routes PDFs to pdfplumber, which finds no text layer. `open_notebook/graphs/source.py` then raises:

```python
if not processed.content or not processed.content.strip():
    raise ValueError("Could not extract any text content from this source. ...")
```

`ValueError` is in the command's `stop_on` list, so despite `max_attempts: 15` the job is marked failed with no retry. No empty source is persisted and nothing is embedded.

`_usable_engine()` separately guards against a stored engine choice whose runtime is absent: it falls back to `auto` and logs, rather than handing content-core an engine that is not installed.

### Docling is a library, not a service

It is imported in-process by content-core inside the Open Notebook container. There is no daemon, no port and no compose entry. Enabling OCR means a derived image plus `OPEN_NOTEBOOK_ENABLE_DOCLING=true` and `HF_HUB_OFFLINE=1`.

The entrypoint probes the venv with `has_module docling` on every boot and skips the install when the package is present, which is what makes the baked image work offline. The venv lives in the image layer, not on the data volume, so this cannot be solved by caching.

`GET /api/capabilities` probes actual importability rather than trusting the enable flag — it is the honest answer about whether OCR is live. `docling_ocr` defaults to `True` in `ContentSettings`, so OCR is on once Docling is present.

### Provisioning is API-driven, not env-driven

Credentials, model records and default assignments all live in the database, and nothing seeds them at startup. The provider environment variables still work but are documented upstream as a deprecated fallback that new automation should not be built on; a declarative replacement is being designed in Discussion #765.

`create_credential_from_env()` reads only `OPENAI_COMPATIBLE_BASE_URL`, so a single credential cannot serve two vLLM endpoints. Create one credential per endpoint through `POST /api/credentials` instead, then one model record each, then `PUT /api/models/defaults`.

### No user accounts

`OPEN_NOTEBOOK_PASSWORD` is a single shared password; unset means auth is disabled entirely. There is no users table and no ownership field on any record — grepping the migrations for `owner`, `user_id` and `created_by` returns nothing. This is a documented product decision ([PDR-001](../7-DEVELOPMENT/decisions/PDR-001-single-user-first.md)), not an oversight. Separate research means separate instances.

---

## Gotchas that cost time

| Gotcha | Consequence |
|---|---|
| `pull_policy: always` in every shipped compose file | Fails immediately with no registry access |
| `HF_HOME` defaults inside `notebook_data` | The dry-run teardown deletes the Docling model cache you just built |
| Hostnames in model base URLs | `url_validation.py` hard-fails on unresolvable names; IP literals skip DNS entirely |
| The entrypoint logs to stdout | Command substitution captures the banner; use `--entrypoint` to read the venv directly |
| Stock image with `ENABLE_DOCLING=true` | Install fails, logs a warning, boots without OCR — no crash to alert you |
| SELinux without `:z` on bind mounts | SurrealDB fails to start with errors that read like a disk fault |
| firewalld and the `docker0` bridge | The container cannot reach vLLM on the host |

---

## Status

As of 2026-09-21, the runbook has not been executed end to end — it is derived from the codebase, not from a completed deployment. The derived-image build in step 3b is the least-exercised part.

Upstream issue #374 tracks pre-baking the opt-in extras via a build arg. If that ships, the derived image and the maintenance burden that comes with it go away.

---

## Related

- [Air-Gapped Deployment Runbook](airgapped-deployment.md)
- [ADR-007: opt-in runtimes](../7-DEVELOPMENT/decisions/ADR-007-optin-runtimes.md)
- [PDR-001: single-user first](../7-DEVELOPMENT/decisions/PDR-001-single-user-first.md)
- [Reverse proxy configuration](../5-CONFIGURATION/reverse-proxy.md)
