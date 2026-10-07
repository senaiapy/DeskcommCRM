## Purpose
The organization's product catalog: priced items (code, name, price, optional cost, stock control, photos) used by proposal items and by the AI agent's product search. Page `app/app/products` (list with server-side search and pagination, editing for manager+). API `app/api/v1/products` (GET/POST), `app/api/v1/products/[id]` (PATCH/DELETE), `app/api/v1/products/[id]/fotos` (POST/PUT), `app/api/v1/products/import` (POST CSV, GET). Logic in `lib/catalogo/*` (`moeda-da-org.ts`, `planilha.ts`, `busca.ts` token search used by `lib/mcp/tools/comercio.ts`, `busca-da-tela.ts`, `fotos.ts`). Table `catalog_products` (`preco_cents`, `custo_cents`, `moeda`, `fotos text[]`); photos in the private storage bucket `catalog-photos`.

## ADDED Requirements

### Requirement: Catalog table with money as cents and ISO currency
`catalog_products` SHALL store `preco_cents bigint not null >= 0`, optional `custo_cents >= 0`, `moeda` matching `^[A-Z]{3}$`, `quantidade >= 0`, and a unique `(organization_id, codigo)` index `catalog_products_org_codigo_key`.

#### Scenario: Negative price
- **WHEN** a row is written with `preco_cents = -1`
- **THEN** Postgres rejects it with constraint `catalog_products_preco_nao_negativo`

#### Scenario: Duplicate code on create
- **WHEN** `POST /api/v1/products` uses a `codigo` already present in the organization
- **THEN** the response is 409 `conflict`

### Requirement: Read for members, write for managers
`GET /api/v1/products` SHALL require role `viewer`, while `POST /api/v1/products`, `PATCH`/`DELETE /api/v1/products/[id]`, `/fotos` and `/import` SHALL require role `manager`, mirrored by RLS policies `catalog_products_insert`, `catalog_products_write` and `catalog_products_delete` using `fn_role_at_least(organization_id, 'manager')`.

#### Scenario: Agent tries to create
- **WHEN** a user with role `agent` calls `POST /api/v1/products`
- **THEN** the response is 403 `forbidden_role`

#### Scenario: Viewer lists
- **WHEN** a viewer calls `GET /api/v1/products`
- **THEN** the response is 200 with up to 500 products of the active organization ordered active first, then by name

### Requirement: Currency comes from the organization
Product writes (`POST /api/v1/products` and `POST /api/v1/products/import`) SHALL set `moeda` from `organizations.currency` via `moedaDaOrganizacao`, falling back to `BRL`, and never from the request body.

#### Scenario: Body carries a currency
- **WHEN** a manager posts a product with `moeda: "USD"` in an organization whose currency is `EUR`
- **THEN** the stored row has `moeda = 'EUR'`, `origem = 'manual'`, and the response is 201

### Requirement: Search and optional pagination
`GET /api/v1/products` SHALL filter by `?busca=` on the server and, when `?pagina=N` is valid, return 50 items per page with `meta.total`, `meta.pagina`, `meta.por_pagina` and `meta.has_more`.

#### Scenario: Term below the minimum
- **WHEN** `busca` is non-empty but yields no usable filter (e.g. a single character or only punctuation)
- **THEN** the response is 200 with an empty list and the database is not queried for products

#### Scenario: Page beyond the end
- **WHEN** `pagina` exceeds the last page
- **THEN** the response is 200 with an empty list and `meta.total` set to the current count

### Requirement: CSV import upserts by code
`POST /api/v1/products/import` SHALL accept a `.csv` file in the multipart field `file`, decode cp1252/UTF-8, and upsert rows with `onConflict: "organization_id,codigo"` without overwriting `descricao`, `imagem_url` or `ativo`, returning `total_linhas`, `criados`, `atualizados`, `erros` and `colunas_ignoradas`.

#### Scenario: Non-CSV file
- **WHEN** the uploaded file is not a CSV
- **THEN** the response is 422 `validation_failed`

#### Scenario: Re-import updates prices
- **WHEN** the same sheet is imported again with changed prices
- **THEN** existing rows matched by `codigo` are updated and `atualizados` counts them, with no duplicated products

### Requirement: Product photos
`POST /api/v1/products/[id]/fotos` SHALL accept JPEG or PNG files up to 5 MB, store them under `<organization_id>/<product_id>/` in bucket `catalog-photos`, and keep at most 5 paths in `catalog_products.fotos` (constraint `catalog_products_fotos_no_maximo_5`).

#### Scenario: Sixth photo
- **WHEN** a product already has 5 photos
- **THEN** the response is 422 `validation_failed`

#### Scenario: Wrong media type or size
- **WHEN** the file is a GIF or exceeds 5 MB
- **THEN** the response is 415 `unsupported_media_type` or 413 `payload_too_large` respectively

### Requirement: Hard delete removes photos
`DELETE /api/v1/products/[id]` SHALL delete the `catalog_products` row of the active organization, remove its photos from storage, and write audit action `catalog_product.deleted`.

#### Scenario: Product of another organization
- **WHEN** the id does not exist in the active organization
- **THEN** the response is 404 `not_found`

### Requirement: Token-based product search for the agent
The agent catalog search (`lib/catalogo/busca.ts`) SHALL match word tokens fuzzily (prefix/trigram over the `catalog_products_nome_trgm` GIN index) and number tokens exactly, so that a number the customer typed that is absent from the product eliminates it.

#### Scenario: Storage variant
- **WHEN** the customer asks for "iphone 15 256" and the catalog has "iPhone 15 Pro 128GB" and "iPhone 15 Pro 256GB"
- **THEN** only the 256GB product is returned
