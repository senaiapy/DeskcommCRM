## MODIFIED Requirements

### Requirement: DELETE archives a pipeline unless it is a clean accidental one
`DELETE /api/v1/pipelines/[id]` SHALL set `crm_pipelines.is_archived = true` by default, and SHALL only hard-delete when `?definitivo=1` is passed and the pipeline has zero `crm_leads`, refusing with 422 `unprocessable_entity` (with `details.negocios`, `details.fontes_de_webhook`, `details.automacoes`) when it is the only active pipeline, the default pipeline, the target of a `webhook_sources.default_pipeline_id`, or referenced by an active automation rule; the hard-delete refusal for a pipeline with deals SHALL advise archiving when the pipeline is active and unarchiving and resolving the deals when it is already archived (`podeExcluirDeVez` in `lib/pipelines/pipeline-editing.ts`).

#### Scenario: Archiving the default pipeline
- **WHEN** a manager deletes the pipeline that has `is_default = true`
- **THEN** the response is 422 `unprocessable_entity` and the row is unchanged

#### Scenario: Hard delete of a pipeline with deals
- **WHEN** `DELETE /api/v1/pipelines/[id]?definitivo=1` targets a pipeline that has at least one `crm_leads` row
- **THEN** the response is 422 with `details.negocios` greater than zero and nothing is deleted

#### Scenario: Plain archive
- **WHEN** a non-default, non-sole pipeline with no webhook source or active automation is deleted without `definitivo`
- **THEN** the row stays with `is_archived = true` and an audit `pipeline.archived` is written

#### Scenario: Hard delete of an archived pipeline with deals
- **WHEN** `DELETE /api/v1/pipelines/[id]?definitivo=1` targets a pipeline with `is_archived = true` and 2 leads
- **THEN** the response is 422, nothing is deleted, and the message tells to take the pipeline out of the archive and resolve the deals instead of archiving it again

## ADDED Requirements

### Requirement: Drag position is computed against the whole stage
When a card is dropped on a filtered board, `KanbanBoard` SHALL take the card above from the visible list and the card below from `proximoNaEtapaInteira()` (`lib/kanban/vizinho-na-etapa.ts`) over the unfiltered board cache `chaveDoQuadro(pipelineId)`, so that the `midpoint` sent to `POST /api/v1/leads/[id]/move` never equals the `position_in_stage` of a hidden card.

#### Scenario: Drop at the end of a filtered column
- **WHEN** a filter hides a card at position 3000 that sits after the last visible card at position 2000, and a card is dropped at the end of the visible column
- **THEN** the new `position_in_stage` is 2500, not 3000

#### Scenario: Board without filter
- **WHEN** the unfiltered cache is missing
- **THEN** the visible card below is used as the lower neighbour
