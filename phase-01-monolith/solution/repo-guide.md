# Repo Guide — `phase-01-monolith/`

> Phase 01 deliverable #3. A map of every top-level file and folder under
> `phase-01-monolith/`, so a newcomer knows where to look before changing anything.

## Orientation

`phase-01-monolith/` is **MIC**, a legacy **PHP 8.2 + MySQL invoicing monolith** (a fictional
invoicing SaaS for Italian SMBs). It runs as **three containers** orchestrated by
`docker-compose`:

| Container | Port | What it is |
|---|---|---|
| `mic-app` | `8088` | The application: nginx + php-fpm in **one** image, run together by supervisord. Serves both the JSON API and the SPA. |
| `mysql` | `3306` | MySQL 8.0. Schema + seed auto-loaded on first boot. |
| `adminer` | `8082` | Web DB browser, for inspecting the data by hand. |

The frontend is a **vanilla-JS single-page app** (no build step, no framework). The backend has
**no Composer / no dependencies** — a hand-rolled autoloader and router. Everything you need to
run it is in `README.md`.

The one thing to internalize up front: **all business entities (clienti, articoli, ordini,
fatture, …) live in two generic tables** — `business_data` and `business_relations`. The meaning
of each column depends on `record_type`. That design is intentional (it's the anti-pattern this
training is built around), and it's why the code is shaped the way it is.

## Top-level map

| Entry | What it is / does |
|---|---|
| [`README.md`](./README.md) | The Phase-01 mission, run instructions (`docker compose up --build`), the five deliverables, success criteria, and exit questions. **Start here.** |
| [`README-IT.md`](./README-IT.md) | Italian mirror of `README.md`. Same content, for Italian readers. |
| [`Dockerfile`](./Dockerfile) | Builds the `mic-app` image from `php:8.2-fpm-alpine`: adds nginx + supervisor, installs the `pdo`/`pdo_mysql` extensions, generates the supervisord config (runs php-fpm **and** nginx in the same container), copies `php-app/` to `/app`, and sets a healthcheck that curls `/api/dashboard/kpi`. |
| [`docker-compose.yml`](./docker-compose.yml) | Defines the three services above, their ports, the `DB_*` env vars passed to the app, the `mic_db_data` volume, and the schema/seed init mounts. MySQL is gated by a healthcheck so the app waits for it. |
| [`nginx.conf`](./nginx.conf) | Reverse-proxy / web server config inside `mic-app`. Serves `/css/` and `/js/` as static files; routes **everything else** (API and SPA) to `index.php` over fastcgi to php-fpm on `127.0.0.1:9000`. |
| [`openapi.yaml`](./openapi.yaml) | OpenAPI 3.1 contract for the legacy `/api/*` routes. Documents the runtime shape — unversioned paths, the `{ "data": … }` / `{ "data": …, "meta": … }` response envelopes, and `{ "error": "…" }` errors. These are the routes that later phases migrate. |
| [`database/`](./database/) | DB bootstrap, run by MySQL automatically on first boot (alphabetically): `schema.sql` creates the two generic tables; `seed.sql` loads the demo dataset (~6.6k lines). See below. |
| [`php-app/`](./php-app/) | The application code itself — backend (PHP) and frontend (SPA). See breakdown below. |

## `database/`

| File | What it is |
|---|---|
| [`schema.sql`](./database/schema.sql) | Creates the `mic` database and the **two** tables. Its header comments are the single best reference for what each generic column means per `record_type` (e.g. for `cliente`: `code`=P.IVA, `name`=ragione sociale, `text_1`=email, …). Read these comments before trusting any column. |
| [`seed.sql`](./database/seed.sql) | Demo data: clients, articles, orders, invoices, etc., all as `business_data` rows plus `business_relations` linking them. |

**`business_data`** — one row per entity of any of 16+ types. Generic columns:
`code`, `name`, `description`, `parent_id`/`parent_type`, `amount_1..4`, `date_1..3`,
`text_1..5`, `status`, and a `payload_json` catch-all. `record_type` disambiguates them.

**`business_relations`** — one row per "soft" link between two `business_data` rows:
`source_id`, `target_id`, `relation_type` (e.g. `cliente_di_ordine`, `fattura_di_ordine`,
`articolo_in_riga_ordine`), plus an optional `amount` and `metadata`. This is how the app fakes
foreign keys and N-N relationships.

## `php-app/`

### Backend (PHP)

| Path | Role |
|---|---|
| [`index.php`](./php-app/index.php) | **Front controller.** Registers a PSR-style autoloader (no Composer), parses the request, serves static assets as a dev fallback, then either dispatches `/api/*` to the router or returns the SPA shell. The whole API route table is declared here: a `$crud` helper wires the five CRUD routes per controller, plus per-domain extras like `/api/orders/:id/righe`, `/api/invoices/:id/invia-sdi`, `/api/customers/:id/invoices`. Finally it JSON-encodes the result and maps errors/`null` to 500/404. |
| [`src/Router.php`](./php-app/src/Router.php) | Tiny regex router. Turns `/api/foo/:id` patterns into regexes, matches method + path, passes captured params to the handler, and returns `[status, body]` (or `null` → 404). |
| [`src/Database.php`](./php-app/src/Database.php) | Singleton-ish PDO factory. Reads `DB_HOST/PORT/NAME/USER/PASS` from env (with sane defaults), and **retries for ~15s on cold start** so the app survives MySQL still booting. |
| [`src/Repository.php`](./php-app/src/Repository.php) | **The heart of the monolith — and the anti-pattern made concrete.** Generic CRUD over `business_data` (`findByType`, `findById`, `create`, `update`, `delete`) plus relation helpers (`relate`, `targetIdOf`, `relationsFrom/To`, …) over `business_relations`. Every controller routes through here; the meaning of `amount_N`/`text_N` lives in the controllers, not the schema. |
| [`src/Models/BusinessData.php`](./php-app/src/Models/BusinessData.php) | Generic DTO for a `business_data` row (`fromRow` / `toArray`). Deliberately untyped per-domain — typing is the caller's job. |
| [`src/Models/BusinessRelation.php`](./php-app/src/Models/BusinessRelation.php) | Generic DTO for a `business_relations` row. |
| [`src/Controllers/BaseController.php`](./php-app/src/Controllers/BaseController.php) | Shared CRUD plumbing: generic `index` (with search/filter/pagination), `show`, `create`, `update`, `destroy`. Subclasses implement `type()` (the `record_type` slug) and override `toDto()` / `fromPayload()` to translate the generic columns to and from friendly field names. |
| `src/Controllers/*Controller.php` | **17 concrete controllers, one per domain** (see table below). Each maps the generic columns to a readable DTO and adds any domain-specific routes. Some also wire relations on write — e.g. [`InvoiceController`](./php-app/src/Controllers/InvoiceController.php) creates `cliente_di_fattura` / `fattura_di_ordine` links and has a mock `invia-sdi` action. |

The concrete controllers and their `record_type`:

| Controller | `record_type` | Controller | `record_type` |
|---|---|---|---|
| `CustomerController` | `cliente` | `MagazzinoController` | `magazzino` |
| `SupplierController` | `fornitore` | `MovimentoController` | `movimento` |
| `ArticleController` | `articolo` | `InvoiceController` | `fattura` |
| `CategoryController` | `categoria` | `NotaCreditoController` | `nota_credito` |
| `IvaController` | `aliquota_iva` | `PagamentoController` | `pagamento` |
| `ListinoController` | `listino` | `UserController` | `utente` |
| `ScontoController` | `sconto` | `AuditController` | `audit_log` |
| `OrderController` | `ordine` | `DashboardController` | *(aggregates KPIs; no single type)* |
| `AgenteController` | `agente` | | |

### Frontend (`public/`)

A no-build SPA served from `index.php` / nginx.

| Path | Role |
|---|---|
| [`public/index.html`](./php-app/public/index.html) | The SPA shell: sidebar nav (Dashboard, Articoli, Clienti, Fornitori, Listini, Ordini, Fatture, Magazzino, Sconti, Agenti, plus Impostazioni), topbar, and the `<script>` tags. Pulls Chart.js from a CDN and calls `MIC.boot()`. |
| [`public/css/style.css`](./php-app/public/css/style.css) | All styling. |
| [`public/js/app.js`](./php-app/public/js/app.js) | The SPA core (`window.MIC`): hash-based router (`#/customers`, …), `fetch` wrapper with toast-on-error, and shared helpers — `listView` (paginated table), `modal`, `confirmDialog`, `buildForm`, money/date formatters. Views register themselves here. |
| `public/js/<domain>.js` | One file per screen, each registering a view and calling the API: `dashboard`, `articles`, `customers`, `suppliers`, `listini`, `orders`, `invoices`, `magazzino`, `sconti`, `agenti`, and `settings` (categorie / aliquote IVA / utenti). |

## How a request flows

```
browser
  → nginx (nginx.conf): static? serve from public/. else →
  → index.php (front controller): /api/* → Router; else → SPA shell (index.html)
  → Router.php: match method + path, extract :params
  → <Domain>Controller (extends BaseController): map payload ↔ columns
  → Repository.php: SQL over business_data / business_relations
  → Database.php (PDO) → MySQL
  ← JSON envelope ({data} or {data, meta}) back up the same path
```

## "Where do I look to…"

| I want to… | Look at |
|---|---|
| Run / stop the app | [`README.md`](./README.md), [`docker-compose.yml`](./docker-compose.yml) |
| Add or change an API route | [`php-app/index.php`](./php-app/index.php) (route table) + the relevant controller |
| Understand what a column actually means | [`database/schema.sql`](./database/schema.sql) header comments |
| Change how data is stored / queried | [`php-app/src/Repository.php`](./php-app/src/Repository.php) (+ `schema.sql`) |
| Understand or change a screen | `php-app/public/js/<domain>.js` + the matching `*Controller.php` |
| See the API contract | [`openapi.yaml`](./openapi.yaml) |
| Inspect the live data | Adminer at `http://localhost:8082` |
| Trace one request end-to-end | `index.php` → `Router.php` → a controller → `Repository.php` → `Database.php` |
