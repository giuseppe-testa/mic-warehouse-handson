# MIC Coupling Map

> **TL;DR:** All data lives in two God Tables (`business_data` + `business_relations`).
> Eight named cross-domain couplings exist. The hardest to break: Dashboard fan-out (C7),
> Order → Catalog price join (C1), and Invoice → Order link (C4).

> Produced from static analysis of `database/schema.sql` and `php-app/src/`.
> Every claim cites the source (file:line or column name).

---

## 1. Storage Layer

MIC stores **all** business entities in two tables, using a type-discriminator pattern
(sometimes called "God Table" or "Single Table Inheritance at scale").

### `business_data` — universal entity table

| Column | Role |
|---|---|
| `record_type` | Discriminator: `cliente`, `fornitore`, `contatto`, `articolo`, `categoria`, `aliquota_iva`, `listino`, `voce_listino`, `sconto`, `magazzino`, `movimento`, `agente`, `ordine`, `riga_ordine`, `fattura`, `nota_credito`, `pagamento`, `utente`, `audit_log` |
| `code` | SKU / P.IVA / invoice number / listino code — meaning depends on `record_type` |
| `amount_1..4` | Price, total, percentage, quantity — meaning depends on `record_type` |
| `text_1..5` | Email, address, IVA code, etc. — meaning depends on `record_type` |
| `date_1..3` | Creation date, expiry, validity — meaning depends on `record_type` |
| `parent_id` / `parent_type` | Polymorphic parent (e.g. `riga_ordine` → `ordine`) |
| `payload_json` | Overflow: fields that don't fit the generic columns |
| `status` | Lifecycle state: `attivo`, `bozza`, `confermato`, `inviato`, `pagato`, `annullato`, `scaduto` |

Source: `database/schema.sql` lines 29–72.

### `business_relations` — universal relation table

All N-N and "soft" 1-N links between entities live here, discriminated by `relation_type`.
There are **no real foreign-key constraints** between domains — referential integrity is
enforced only at the PHP application layer.

| Column | Role |
|---|---|
| `source_id` | ID of the "from" entity in `business_data` |
| `target_id` | ID of the "to" entity in `business_data` |
| `relation_type` | Discriminator string (e.g. `cliente_di_ordine`, `articolo_in_riga_ordine`) |
| `amount` | Quantity, unit price, discount %, payment amount — meaning depends on `relation_type` |
| `metadata` | JSON overflow for extra relation attributes |

Source: `database/schema.sql` lines 92–104.

---

## 2. Domain Clusters

The 19 `record_type` values map onto 7 functional domains:

| Domain | `record_type` members | Controller(s) |
|---|---|---|
| **Customer / Contact** | `cliente`, `fornitore`, `contatto` | CustomerController, SupplierController |
| **Catalog** | `articolo`, `categoria` | ArticleController, CategoryController |
| **Pricing & Tax** | `aliquota_iva`, `listino`, `voce_listino`, `sconto` | IvaController, ListinoController, ScontoController |
| **Order Management** | `ordine`, `riga_ordine` | OrderController |
| **Invoicing** | `fattura`, `nota_credito`, `pagamento` | InvoiceController, NotaCreditoController, PagamentoController |
| **Warehouse** | `magazzino`, `movimento` | MagazzinoController, MovimentoController |
| **Sales & Identity** | `agente`, `utente`, `audit_log` | AgenteController, UserController, AuditController |

> Note: `DashboardController` does not own a domain — it reads across **all** domains
> in a single method (`kpi()`). See Section 4, coupling C7.

---

## 3. Read/Write Map

"WRITES" = creates or updates records of this type. "READS ACROSS" = reads records
owned by another domain. Evidence is file:line.

| Domain | WRITES (record_type) | WRITES (relation_type) | READS ACROSS |
|---|---|---|---|
| **Order Management** | `ordine`, `riga_ordine` | `cliente_di_ordine`, `agente_di_ordine`, `listino_di_ordine`, `articolo_in_riga_ordine`, `iva_di_riga`, `sconto_su_ordine` | `cliente` (OrderController:16–18), `agente` (OrderController:18), `listino` (OrderController:17), `articolo.amount_1+text_2` (OrderController:132–138), `aliquota_iva` (OrderController:139–142) |
| **Invoicing** | `fattura`, `nota_credito`, `pagamento` | `cliente_di_fattura`, `fattura_di_ordine`, `pagamento_di_fattura` | `cliente` (InvoiceController:16), `ordine` (InvoiceController:17) |
| **Warehouse** | `magazzino`, `movimento` | `movimento_di_articolo`, `movimento_in_magazzino` | `articolo` (MagazzinoController:50–67) |
| **Dashboard** | _(none — read-only)_ | _(none)_ | `fattura`, `ordine`, `articolo`, `cliente`, `audit_log`, `utente` (DashboardController:20–106) |
| **Catalog** | `articolo`, `categoria` | `categoria_di_articolo` | `aliquota_iva` (embedded as `text_2` code in `articolo`) |
| **Pricing & Tax** | `aliquota_iva`, `listino`, `voce_listino`, `sconto` | `voce_di_listino`, `articolo_di_voce` | _(none)_ |
| **Customer / Contact** | `cliente`, `fornitore`, `contatto` | _(none)_ | _(none)_ |
| **Sales & Identity** | `agente`, `utente`, `audit_log` | _(none)_ | _(none)_ |

---

## 4. Cross-Domain Couplings

Each coupling is a dependency that a future Bounded Context extraction must break.
Named by convention: **SOURCE → TARGET** (direction of data flow / dependency).

---

### C1 — Order Management → Catalog (price + IVA code)

**What:** When adding a line (`riga_ordine`) to an order, `OrderController::addRiga()`
reads `articolo.amount_1` (prezzo_listino) and `articolo.text_2` (iva_default code)
directly from the shared `business_data` table, then queries `aliquota_iva` by that code
to compute the VAT amount.

**Evidence:**
- `OrderController.php:132–138` — `$this->repo->findById($art_id)` fetches the full
  `articolo` row and uses `$articolo->amount1` and `$articolo->text2`
- `OrderController.php:139–142` — raw query `WHERE record_type='aliquota_iva' AND code=?`

**Extraction cost:** HIGH. Extracting Orders into a microservice requires either:
(a) a Catalog API to fetch article price and IVA code at order-line creation time, or
(b) denormalising price into the order line at write time and breaking the live join.

---

### C2 — Order Management → Pricing (listino + sconto)

**What:** An order can be associated with a `listino` (price list) and a `sconto` (discount)
via `business_relations`. `OrderController::create()` wires these at creation time and
`toDto()` resolves them back on read.

**Evidence:**
- `OrderController.php:77–80` — `relate($b->id, $listino_id, 'listino_di_ordine')`
- `OrderController.php:16–18` — `lookupRelTarget($id, 'listino_di_ordine')`
- `database/schema.sql:89` — `relation_type='sconto_su_ordine'`

**Extraction cost:** MEDIUM. The listino and sconto IDs need to be either replicated
into the Order service or looked up via a Pricing API.

---

### C3 — Order Management → Customer (client identity)

**What:** Every order must have a `cliente_id` (required field). OrderController reads
the customer's name for DTO output on every GET request.

**Evidence:**
- `OrderController.php:72–73` — validation `if (!$cliente_id) return [400, ...]`
- `OrderController.php:14–17` — `lookupRelTarget($id, 'cliente_di_ordine')`

**Extraction cost:** MEDIUM. Extracting Orders requires a Customer reference (ID + name)
to be available at order read time — either cached in the order row or fetched via API.

---

### C4 — Invoicing → Order Management (invoice linked to order)

**What:** An invoice (`fattura`) can be linked to an order via `fattura_di_ordine`.
`InvoiceController` reads the order's code on every invoice DTO.

**Evidence:**
- `InvoiceController.php:17` — `lookupRelTarget($id, 'fattura_di_ordine')`
- `InvoiceController.php:77` — `relate($b->id, $ordine_id, 'fattura_di_ordine')`

**Extraction cost:** HIGH. Extracting Invoicing requires coordination with the Order
service (at minimum: order existence check + order number lookup).

---

### C5 — Invoicing → Customer (client identity)

**What:** Same pattern as C3 but for invoices. Every `fattura` holds a `cliente_di_fattura`
relation and the customer name is resolved on read.

**Evidence:**
- `InvoiceController.php:16` — `lookupRelTarget($id, 'cliente_di_fattura')`
- `InvoiceController.php:76` — `relate($b->id, $cliente_id, 'cliente_di_fattura')`

**Extraction cost:** MEDIUM (same as C3 — needs Customer API or denormalisation).

---

### C6 — Warehouse → Catalog (stock movements by article)

**What:** `MagazzinoController::giacenze()` computes stock levels by joining `movimento`
records back to `articolo` records through `business_relations`. Warehouse stock is
meaningless without the Catalog's article identity.

**Evidence:**
- `MagazzinoController.php:50–67` — 5-table join across `magazzino → movimento →
  articolo` via `movimento_in_magazzino` + `movimento_di_articolo` relation types

**Extraction cost:** HIGH. Extracting Warehouse requires either replicating article
metadata into the Warehouse service or calling a Catalog API for every stock query.
The `giacenze` query mixes Warehouse and Catalog data in a single SQL join — this join
must become an inter-service call or an event-driven materialised view.

---

### C7 — Dashboard → All Domains (read-only fan-out)

**What:** `DashboardController::kpi()` issues raw SQL queries against `fattura`, `ordine`,
`articolo`, `cliente`, `audit_log` in a single PHP method. It is a global read coupling.

**Evidence:**
- `DashboardController.php:20–106` — seven separate rawOne/rawAll calls spanning
  Invoicing, Order Management, Catalog, Customer, and Identity domains

**Extraction cost:** HIGH for microservices. The dashboard would need to aggregate data
from 5+ services. Mitigation: introduce a dedicated read model / reporting service that
subscribes to domain events and keeps its own denormalised summary table.

---

### C8 — Catalog → Tax (IVA code embedded in article)

**What:** `articolo` stores the default IVA code as a plain string in `text_2`
(`iva_default`). This is not a foreign key — it is a magic string that OrderController
uses to look up the `aliquota_iva` record at order-line creation time (see C1).

**Evidence:**
- `ArticleController.php:30` — `'iva_default' => $r['text_2']`
- `ArticleController.php:48` — `'text_2' => $b['iva_default'] ?? 'IVA22'`
- `OrderController.php:137–142` — lookup `aliquota_iva` WHERE `code=?`

**Extraction cost:** LOW to MEDIUM. The fix is to resolve and embed the IVA percentage
at article-write time, or to make Catalog the authoritative source of IVA codes and
expose them via API.

---

## 5. Extraction Targets

Ranked by extraction readiness (fewest inbound couplings = easiest to extract first).

| Rank | Domain | Inbound couplings | Outbound couplings | Extraction readiness |
|---|---|---|---|---|
| 1 | **Pricing & Tax** | none | none | Easiest — no domain reads its data except Order and Catalog |
| 2 | **Customer / Contact** | C3, C5 (others read its data) | none | Self-contained writes; only needs to expose a read API |
| 3 | **Sales & Identity** | none | none | Agente is only referenced by Order; utente/audit_log are standalone |
| 4 | **Catalog** | C1, C6, C8 (others read it) | C8 (depends on Tax code) | High fan-in but no writes from other domains |
| 5 | **Warehouse** | none | C6 (reads Catalog) | Clean domain but `giacenze` query couples it tightly to Catalog |
| 6 | **Invoicing** | none | C4, C5 (reads Order + Customer) | Needs Order and Customer APIs before extraction |
| 7 | **Order Management** | C2, C3 (others reference it) | C1, C2, C3 (reads Catalog, Pricing, Customer) | Most coupled — extract last or alongside its dependencies |

> **Recommended first extraction: Warehouse (Phase 07 target per MagazzinoController comment).**
> The Warehouse → Catalog coupling (C6) is the single dependency to break: replace the
> multi-join SQL with a Catalog API call (or event-driven article replication) and the
> Warehouse BC becomes self-contained.