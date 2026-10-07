## Purpose
The organization knowledge base used for retrieval-augmented answers: knowledge sources (`ai_knowledge_sources`, `ai_faq_items`), versioned chunk indexes (`ai_knowledge_versions`, `ai_chunks` with 1536-dimension embeddings), the `rag-indexer` event handler, the human search endpoint, the embedding-provider switch and the daily `kb-conversations-batch` ingestion of anonymized conversations. Agent versions choose which sources they read via `ai_agent_versions.knowledge_source_ids` (covered by `ai-agents-runtime`); embedding credentials are stored as described in `ai-credentials-budget`.

## ADDED Requirements

### Requirement: Creating or editing a source emits a reindex event
`POST /api/v1/ai/knowledge/sources` and `PATCH /api/v1/ai/knowledge/sources/{id}` SHALL require role `manager`, write `ai_knowledge_sources` (and `ai_faq_items` for FAQ) under the session `organization_id`, answer 404 `not_found` for an `agent_id` outside the organization, and emit `knowledge_source.updated` through the RPC `emit_event` with the source id in the payload.

#### Scenario: New FAQ source
- **WHEN** a manager creates a FAQ source with items
- **THEN** the source and its `ai_faq_items` rows are stored and a `knowledge_source.updated` event names the new source

#### Scenario: FAQ items fail to insert
- **WHEN** inserting `ai_faq_items` fails after the source row was created
- **THEN** the source row is deleted and the response is 500 `internal_error`

### Requirement: File uploads are capped at 20 MB
`POST /api/v1/ai/knowledge/sources/upload` SHALL require role `manager`, answer 413 `payload_too_large` for files above 20 MB, answer 422 `unprocessable_entity` when text extraction fails with a user-facing message, and emit `knowledge_source.updated` after registering the source.

#### Scenario: Oversized PDF
- **WHEN** a manager uploads a 25 MB file
- **THEN** the response is 413 `payload_too_large` and no source row is written

### Requirement: Deleting a source is a soft archive
`DELETE /api/v1/ai/knowledge/sources/{id}` SHALL set `status = 'archived'` and `is_active = false` on the source of the session organization instead of deleting rows.

#### Scenario: Archive keeps history
- **WHEN** a manager deletes a source
- **THEN** the response is `{ id, status: 'archived' }` and its chunks and versions remain in the database

### Requirement: The indexer builds a new version per run and activates it only when complete
The `rag-indexer.v1` handler SHALL consume `knowledge_source.updated` and `nuvemshop.product_synced`, take `organization_id` from the event row, skip when `content_hash` is unchanged, `last_index_status = 'success'` and the active version used the same embedding model, otherwise create a new `ai_knowledge_versions` row, upsert one `ai_chunks` row per chunk on `(knowledge_source_id, kb_version_id, position)`, and call `activateVersion` only when every chunk was written.

#### Scenario: One chunk fails to embed
- **WHEN** the embedding call fails on chunk 3 of 10
- **THEN** the new version is marked failed and the previous active version keeps serving searches

#### Scenario: Unchanged content
- **WHEN** a source is reindexed with the same content and model
- **THEN** no new version is created and no embedding call is made

### Requirement: Missing embedding key is a visible, retried state
When no embedding key resolves for the organization, the indexer SHALL set `ai_knowledge_sources.last_index_status = 'sem_credencial'`, open an `agent_inbox_items` row of kind `conhecimento_nao_indexado`, and return the event as `retry` one hour later instead of `skipped`.

#### Scenario: Key added the next day
- **WHEN** an organization without an embedding key adds one after a source was created
- **THEN** the retried event indexes the source without the user re-saving it

### Requirement: Chunking uses paragraph splitting with overlap
`chunkText` SHALL split text by paragraphs, sub-split paragraphs longer than `maxChars` (default 1500) at sentence boundaries, and prepend `overlapChars` (default 200) of the previous chunk to the next.

#### Scenario: Long document
- **WHEN** a 10,000-character document is chunked with defaults
- **THEN** every chunk is at most about 1500 characters and consecutive chunks share up to 200 characters

### Requirement: The embedding family is an organization setting
`PUT /api/v1/ai/knowledge/provedor` SHALL require role `admin`, accept `provedor` in `openai` or `google`, write `organizations.settings.base_de_conhecimento.familia` only when that family has a usable key (422 otherwise), and enqueue all sources for reindex; OpenAI SHALL embed with `openai/text-embedding-3-small` and Google with `google/gemini-embedding-001`, both at 1536 dimensions.

#### Scenario: Switching to Google without a key
- **WHEN** an admin chooses `google` while no Google key is usable
- **THEN** the response is 422 and the stored family does not change

### Requirement: Search compares only chunks of the same embedding model
Knowledge search SHALL call the RPC `fn_buscar_trechos_das_fontes` with `p_organization_id`, `p_source_ids`, `p_embedding`, `p_k`, `p_threshold` and `p_embedding_model`, used both by `POST /api/v1/ai/knowledge/busca` and by the agent tool `search_knowledge`.

#### Scenario: Human search
- **WHEN** a user with role `agent` posts a question to `/api/v1/ai/knowledge/busca`
- **THEN** results come only from the organization's sources indexed with the current embedding model and the search is logged in `knowledge_searches`

### Requirement: Human search is rate limited per user and per organization
`POST /api/v1/ai/knowledge/busca` SHALL allow at most 12 questions per user and 60 per organization per 60-second window and answer 429 `rate_limited` with `Retry-After` and `X-RateLimit-*` headers beyond that.

#### Scenario: Thirteenth question in a minute
- **WHEN** one user sends 13 searches within 60 seconds
- **THEN** the 13th response is 429 `rate_limited` with `Retry-After: 60`

### Requirement: Daily conversation ingestion runs once per operating organization
`GET /api/v1/cron/kb-conversations-batch` SHALL authorize with `INTERNAL_CRON_SECRET` or `INTERNAL_SECRET` (403 `forbidden` otherwise), pick one active agent per organization whose status is operating, ingest the last 24 hours of conversations, and audit `rag.conversations_batch_run` only when something was processed, flagged, skipped or failed.

#### Scenario: Suspended organization
- **WHEN** an organization is suspended
- **THEN** the cron spends no embedding call on it
