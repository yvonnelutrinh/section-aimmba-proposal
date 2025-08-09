
# Yvonne Lu Trinh - Section AI for Programmers Mini-MBA
# Project Overview

**Core idea:** Build a Python tool that turns plain-English business needs into a **proposed analytical data model**. This would include a set of JSON and SQL tables with keys, constraints, indexes, and a brief rationale.

**Problem/opportunity:** Business Intelligence work often stalls on “what tables do we need?” Data engineers spend hours translating stakeholder asks into schemas. This tool accelerates that language → schema step, standardizes design patterns (slowly changing dimensions, fact tables, etc.), and produces reviewable DDL(Data Definition Language) that teams can iterate on quickly.

---
# Use Case & Workflow

**Who:** Business Intelligence analysts, analytics engineers, and data engineers scoping a new feaure.

**Workflow with AI:**
1. **Input:** Stakeholder describes the business need (“We need a monthly revenue dashboard by product and region with discounts and refunds separated.”) + optional company context (existing warehouse, naming conventions, conformed dimensions, OR even existing SQL schema).
2. **RAG (optional):** Tool ingests relevant context (current SQL schemas, glossary) and retrieves it to ground the process.
3. **Generation:** LLM proposes a dimensional model (facts/dimensions), outputs **SQL DDL** (CREATE TABLE…), surrogate keys, indexes, and a short justification.
4. **Validation:** Tool lints SQL, sanity-checks keys, and (optionally) spins up a temp DB (SQLite) to verify constraints compile.
5. **Review & Export:** Users review structured JSON + SQL; export to .sql files or open a PR to a repo.

**How AI helps:** Compresses days of back-and-forth into minutes, enforces conventions, and provides consistent structured outputs that are easy to review and tweak.

# AI Features to Be Implemented

- **Prompt engineering:**  
    Template includes business objective, required metrics/dimensions, naming conventions, and constraints → reduces hallucinations and ensures consistency.
- **Structured outputs:** 
    Force the LLM to return a strict **JSON schema** describing tables, columns, types, nullability, keys, FKs, and indexes. Then renders to SQL, guaranteeing machine-parseable results. (see "Evaluation Frameworks")
- **Retrieval-Augmented Generation (RAG) & vector DB (optional):**  
    Use embeddings to pull relevant **existing SQL schemas/glossaries** so designs align with your warehouse and reuse conformed dimensions.
- **Evaluation frameworks:**  
    Automatic checks (see “Evaluation Strategy”) plus lightweight **Langchain** rubric to score model quality. SQL lints via SQLFluff & compile checks via DuckDB/SQLite.
- **Observability tools:**  
    Track prompts, tokens, latency, cost, and failure modes; capture feedback (“accept/revise/reject”) to fine-tune prompts and model choices over time.

# Technical Approach

**Architecture (high level):**
- **Interface:** start with a CLI; add a simple API endpoint + minimal web UI later.
- **Core pipeline:**
    1. **Pre-process:** Parse user brief; normalize to a canonical prompt format; optionally attach retrieved context chunks.
    2. **LLM call:** Generate structured JSON (tables/columns/keys/constraints) with a **Pydantic** schema.
    3. **Render:** Convert JSON → dialect-specific SQL DDL using **SQLGlot** for dialect mapping.
    4. **Validate:** Lint (**SQLFluff**), compile in **DuckDB/SQLite**; run constraint checks and simple insert tests.
    5. **Package:** Emit `design.json`, `schema.sql`, and a short `README.md` with assumptions and usage notes.

**Key tools & services (Python-first):**
- **LLM API:** OpenAI/Anthropic/other provider (choose model with strong function calling/JSON mode).
- **Embeddings & Vector store (optional):** text-embedding model + **FAISS** (local) or **Pinecone**  for RAG.
- **Schema enforcement:** **Pydantic** (define the expected JSON structure); **Guardrails** or model’s native JSON mode as an extra safety net.
- **SQL tooling:** **SQLGlot** (dialects), **SQLFluff** (lint), **DuckDB**/**SQLite** (compile & quick tests).
- **API & Dev UX:** **Typer** (CLI), **Cursor** (dev), **pre-commit** hooks.
- **Observability:** **Langchain** traces; simple **PostHog**/**OpenPanel** for usage; log to **SQLite**/**Postgres**.

# Example Prompts & Expected Outputs

**Prompt 1:**  
“**Design a schema JSON format for the following business requirements:**
 A monthly revenue dashboard. We need revenue, discounts, refunds, and net revenue by product, and sales channel. 

**Database:**
Postgres

**Include a brief rationale for your design**
”


**Expected structured output → rendered JSON(attached design.json) as well as Rationale text:**

```json
{
  "database": "postgres",
  "schema_version": "1.0",
  "entities": [
    {
      "name": "product",
      "description": "Catalog of sellable products.",
      "columns": [
        {"name": "product_id", "type": "uuid", "pk": true, "nullable": false, "default": "gen_random_uuid()"},
        {"name": "sku", "type": "text", "unique": true, "nullable": false},
        {"name": "name", "type": "text", "nullable": false},
        {"name": "active", "type": "boolean", "nullable": false, "default": true},
        {"name": "created_at", "type": "timestamptz", "nullable": false, "default": "now()"}
      ],
      "indexes": ["sku"]
    },
    {
      "name": "sales_channel",
      "description": "Where the order was sold (e.g., web, retail, marketplace).",
      "columns": [
        {"name": "channel_id", "type": "smallserial", "pk": true, "nullable": false},
        {"name": "code", "type": "text", "unique": true, "nullable": false},
        {"name": "name", "type": "text", "nullable": false}
      ],
      "indexes": ["code"]
    },
    {
      "name": "customer_order",
      "description": "Order header; one per checkout.",
      "columns": [
        {"name": "order_id", "type": "uuid", "pk": true, "nullable": false, "default": "gen_random_uuid()"},
        {"name": "order_number", "type": "text", "unique": true, "nullable": false},
        {"name": "channel_id", "type": "smallint", "fk": {"ref": "sales_channel.channel_id"}, "nullable": false},
        {"name": "order_datetime", "type": "timestamptz", "nullable": false},
        {"name": "currency_code", "type": "char(3)", "nullable": false, "default": "USD"},
        {"name": "status", "type": "text", "nullable": false, "check": "status IN ('created','paid','fulfilled','cancelled','refunded','partial_refunded')"}
      ],
      "indexes": ["order_datetime", "channel_id"]
    },
    {
      "name": "order_item",
      "description": "Line items within an order (pre-tax). Monetary amounts are in the order currency.",
      "columns": [
        {"name": "order_item_id", "type": "bigserial", "pk": true, "nullable": false},
        {"name": "order_id", "type": "uuid", "fk": {"ref": "customer_order.order_id"}, "nullable": false},
        {"name": "product_id", "type": "uuid", "fk": {"ref": "product.product_id"}, "nullable": false},
        {"name": "quantity", "type": "integer", "nullable": false, "check": "quantity > 0"},
        {"name": "unit_price", "type": "numeric(12,2)", "nullable": false, "comment": "List price per unit before discounts."},
        {"name": "extended_price", "type": "numeric(12,2)", "nullable": false, "comment": "unit_price * quantity"}
      ],
      "indexes": ["order_id", "product_id"]
    },
    {
      "name": "discount",
      "description": "Discounts applied at item level or order level; order-level discounts can be allocated to items.",
      "columns": [
        {"name": "discount_id", "type": "bigserial", "pk": true, "nullable": false},
        {"name": "order_id", "type": "uuid", "fk": {"ref": "customer_order.order_id"}, "nullable": false},
        {"name": "order_item_id", "type": "bigint", "fk": {"ref": "order_item.order_item_id"}, "nullable": true},
        {"name": "reason", "type": "text", "nullable": true},
        {"name": "amount", "type": "numeric(12,2)", "nullable": false, "check": "amount >= 0"},
        {"name": "currency_code", "type": "char(3)", "nullable": false}
      ],
      "indexes": ["order_id", "order_item_id"]
    },
    {
      "name": "payment",
      "description": "Captured payments for orders (for revenue recognition by payment date if desired).",
      "columns": [
        {"name": "payment_id", "type": "bigserial", "pk": true, "nullable": false},
        {"name": "order_id", "type": "uuid", "fk": {"ref": "customer_order.order_id"}, "nullable": false},
        {"name": "captured_at", "type": "timestamptz", "nullable": false},
        {"name": "amount", "type": "numeric(12,2)", "nullable": false, "check": "amount >= 0"},
        {"name": "currency_code", "type": "char(3)", "nullable": false}
      ],
      "indexes": ["captured_at", "order_id"]
    },
    {
      "name": "refund",
      "description": "Refunds processed against specific order items (preferred) or the order as a whole.",
      "columns": [
        {"name": "refund_id", "type": "bigserial", "pk": true, "nullable": false},
        {"name": "order_id", "type": "uuid", "fk": {"ref": "customer_order.order_id"}, "nullable": false},
        {"name": "order_item_id", "type": "bigint", "fk": {"ref": "order_item.order_item_id"}, "nullable": true},
        {"name": "refunded_at", "type": "timestamptz", "nullable": false},
        {"name": "amount", "type": "numeric(12,2)", "nullable": false, "check": "amount >= 0"},
        {"name": "currency_code", "type": "char(3)", "nullable": false},
        {"name": "reason", "type": "text", "nullable": true}
      ],
      "indexes": ["refunded_at", "order_id", "order_item_id"]
    },
    {
      "name": "fx_rate_daily",
      "description": "Optional: FX rates to convert to a reporting currency.",
      "columns": [
        {"name": "fx_date", "type": "date", "pk": true, "nullable": false},
        {"name": "currency_code", "type": "char(3)", "pk": true, "nullable": false},
        {"name": "rate_to_reporting", "type": "numeric(18,8)", "nullable": false},
        {"name": "reporting_currency", "type": "char(3)", "nullable": false, "default": "USD"}
      ],
      "indexes": ["currency_code", "fx_date"]
    }
}

```
##### Rationale 
- **Dimensional shape for analytics:** Separate **dimensions** (`product`, `sales_channel`) from **facts** (order items, discounts, refunds). This keeps the grain clear and makes rollups fast and reliable.
- **Correct grain:** Revenue and discounts are captured at the **item** level; refunds can attach to items or the whole order (and still roll up by product/channel). This ensures product/channel splits are accurate.
- **Monthly aggregation:** A **materialized view** pre-aggregates by `month × product × channel` so the dashboard stays snappy; index supports month/product/channel filters.
- **Net revenue definition:** `gross_revenue - discounts - refunds`, matching common reporting. All amounts stored as `numeric(12,2)` to avoid float issues.
- **Currency awareness:** Base tables store original currency; optional **FX view** converts to a reporting currency on read.
- **Extensibility:** You can add taxes, regions, or marketplaces as new dimensions without breaking the monthly view; or add additional measures (units sold, AOV) with minimal changes.


## The tool then renders **CREATE TABLE** statements per dialect and a short rationale.

**Prompt 2 (with existing context):**  

“Using the attached data docs, and provided JSON database definition, generate all SQL DDL to create the necessary tables using best practices for 
_database type from previous step (Postgres in this example)_

_pasted confirmed JSON structure from previous step_
”

**Expected SQL output (also attaching a schema.sql file ):** 
```SQL
BEGIN;
-- For gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Optional: work in a dedicated schema (uncomment if you want one)
-- CREATE SCHEMA IF NOT EXISTS commerce;
-- SET search_path = commerce, public;

-- Domain for ISO currency codes (ensures 3 uppercase letters, ignores CHAR(3) padding)
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_type t
    JOIN pg_namespace n ON n.oid = t.typnamespace
    WHERE t.typname = 'iso_currency_code' AND n.nspname = ANY (current_schemas(true))
  ) THEN
    CREATE DOMAIN iso_currency_code AS CHAR(3)
      CHECK ( btrim(VALUE) ~ '^[A-Z]{3}$' );
    COMMENT ON DOMAIN iso_currency_code IS 'Three-letter ISO currency code, uppercase.';
  END IF;
END$$;

-- ===========================================
-- product
-- ===========================================
CREATE TABLE IF NOT EXISTS product (
  product_id    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  sku           text NOT NULL UNIQUE,
  name          text NOT NULL,
  active        boolean NOT NULL DEFAULT true,
  created_at    timestamptz NOT NULL DEFAULT now()
);

COMMENT ON TABLE product IS 'Catalog of sellable products.';
COMMENT ON COLUMN product.product_id IS 'Surrogate key.';
COMMENT ON COLUMN product.sku IS 'Merchant-defined stock keeping unit; unique.';
COMMENT ON COLUMN product.active IS 'Whether the product is currently sellable.';

-- Indexes (unique on sku already created by UNIQUE constraint)
CREATE INDEX IF NOT EXISTS idx_product_sku ON product (sku);

-- ===========================================
-- sales_channel
-- ===========================================
CREATE TABLE IF NOT EXISTS sales_channel (
  channel_id smallserial PRIMARY KEY,
  code       text NOT NULL UNIQUE,
  name       text NOT NULL
);

COMMENT ON TABLE sales_channel IS 'Where the order was sold (e.g., web, retail, marketplace).';
COMMENT ON COLUMN sales_channel.code IS 'Stable identifier (e.g., "WEB", "RETAIL", "AMZ").';

CREATE INDEX IF NOT EXISTS idx_sales_channel_code ON sales_channel (code);

-- ===========================================
-- customer_order
-- ===========================================
CREATE TABLE IF NOT EXISTS customer_order (
  order_id        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number    text NOT NULL UNIQUE,
  channel_id      smallint NOT NULL,
  order_datetime  timestamptz NOT NULL,
  currency_code   iso_currency_code NOT NULL DEFAULT 'USD',
  status          text NOT NULL,
  CONSTRAINT fk_customer_order_channel
    FOREIGN KEY (channel_id) REFERENCES sales_channel(channel_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT chk_customer_order_status
    CHECK (status IN ('created','paid','fulfilled','cancelled','refunded','partial_refunded'))
);

COMMENT ON TABLE customer_order IS 'Order header; one per checkout.';
COMMENT ON COLUMN customer_order.order_number IS 'External/customer-facing order number.';
COMMENT ON COLUMN customer_order.currency_code IS 'Order currency (ISO 4217).';
COMMENT ON COLUMN customer_order.status IS 'Order lifecycle status.';

CREATE INDEX IF NOT EXISTS idx_customer_order_order_datetime ON customer_order (order_datetime);
CREATE INDEX IF NOT EXISTS idx_customer_order_channel_id ON customer_order (channel_id);

-- ===========================================
-- order_item
-- ===========================================
CREATE TABLE IF NOT EXISTS order_item (
  order_item_id   bigserial PRIMARY KEY,
  order_id        uuid NOT NULL,
  product_id      uuid NOT NULL,
  quantity        integer NOT NULL,
  unit_price      numeric(12,2) NOT NULL,
  -- Keep extended_price authoritative via a generated column
  extended_price  numeric(12,2) GENERATED ALWAYS AS (unit_price * quantity) STORED,
  CONSTRAINT fk_order_item_order
    FOREIGN KEY (order_id) REFERENCES customer_order(order_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT fk_order_item_product
    FOREIGN KEY (product_id) REFERENCES product(product_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT chk_order_item_quantity
    CHECK (quantity > 0)
);

COMMENT ON TABLE order_item IS 'Line items within an order (pre-tax). Monetary amounts are in the order currency.';
COMMENT ON COLUMN order_item.unit_price IS 'List price per unit before discounts.';
COMMENT ON COLUMN order_item.extended_price IS 'unit_price * quantity (generated).';

CREATE INDEX IF NOT EXISTS idx_order_item_order_id   ON order_item (order_id);
CREATE INDEX IF NOT EXISTS idx_order_item_product_id ON order_item (product_id);

-- ===========================================
-- discount
-- ===========================================
CREATE TABLE IF NOT EXISTS discount (
  discount_id    bigserial PRIMARY KEY,
  order_id       uuid NOT NULL,
  order_item_id  bigint NULL,
  reason         text NULL,
  amount         numeric(12,2) NOT NULL,
  currency_code  iso_currency_code NOT NULL,
  CONSTRAINT fk_discount_order
    FOREIGN KEY (order_id) REFERENCES customer_order(order_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT fk_discount_order_item
    FOREIGN KEY (order_item_id) REFERENCES order_item(order_item_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT chk_discount_amount
    CHECK (amount >= 0)
);

COMMENT ON TABLE discount IS 'Discounts applied at item level or order level; order-level discounts can be allocated to items.';

CREATE INDEX IF NOT EXISTS idx_discount_order_id       ON discount (order_id);
CREATE INDEX IF NOT EXISTS idx_discount_order_item_id  ON discount (order_item_id);

-- ===========================================
-- payment
-- ===========================================
CREATE TABLE IF NOT EXISTS payment (
  payment_id     bigserial PRIMARY KEY,
  order_id       uuid NOT NULL,
  captured_at    timestamptz NOT NULL,
  amount         numeric(12,2) NOT NULL,
  currency_code  iso_currency_code NOT NULL,
  CONSTRAINT fk_payment_order
    FOREIGN KEY (order_id) REFERENCES customer_order(order_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT chk_payment_amount
    CHECK (amount >= 0)
);

COMMENT ON TABLE payment IS 'Captured payments for orders (for revenue recognition by payment date if desired).';

CREATE INDEX IF NOT EXISTS idx_payment_captured_at ON payment (captured_at);
CREATE INDEX IF NOT EXISTS idx_payment_order_id    ON payment (order_id);

-- ===========================================
-- refund
-- ===========================================
CREATE TABLE IF NOT EXISTS refund (
  refund_id      bigserial PRIMARY KEY,
  order_id       uuid NOT NULL,
  order_item_id  bigint NULL,
  refunded_at    timestamptz NOT NULL,
  amount         numeric(12,2) NOT NULL,
  currency_code  iso_currency_code NOT NULL,
  reason         text NULL,
  CONSTRAINT fk_refund_order
    FOREIGN KEY (order_id) REFERENCES customer_order(order_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT fk_refund_order_item
    FOREIGN KEY (order_item_id) REFERENCES order_item(order_item_id)
      ON UPDATE RESTRICT ON DELETE RESTRICT,
  CONSTRAINT chk_refund_amount
    CHECK (amount >= 0)
);

COMMENT ON TABLE refund IS 'Refunds processed against specific order items (preferred) or the order as a whole.';

CREATE INDEX IF NOT EXISTS idx_refund_refunded_at   ON refund (refunded_at);
CREATE INDEX IF NOT EXISTS idx_refund_order_id      ON refund (order_id);
CREATE INDEX IF NOT EXISTS idx_refund_order_item_id ON refund (order_item_id);

-- ===========================================
-- fx_rate_daily
-- ===========================================
CREATE TABLE IF NOT EXISTS fx_rate_daily (
  fx_date              date NOT NULL,
  currency_code        iso_currency_code NOT NULL,
  rate_to_reporting    numeric(18,8) NOT NULL,
  reporting_currency   iso_currency_code NOT NULL DEFAULT 'USD',
  CONSTRAINT pk_fx_rate_daily PRIMARY KEY (fx_date, currency_code)
);

COMMENT ON TABLE fx_rate_daily IS 'Optional: FX rates to convert to a reporting currency.';
COMMENT ON COLUMN fx_rate_daily.rate_to_reporting IS 'Multiply a value in currency_code by this to get reporting_currency.';

-- Helpful covering index for lookups by (currency_code, fx_date) ranges
CREATE INDEX IF NOT EXISTS idx_fx_rate_daily_ccy_date ON fx_rate_daily (currency_code, fx_date);

COMMIT;
```

# Evaluation Strategy

**Automated checks**

- **SQL validity:** DDL compiles in DuckDB/SQLite (per-dialect compile if available).
- **Lint score:** SQLFluff rules pass (naming, constraints present).
- **Relational integrity:** All Foreign Keys reference defined Primary Keys; PK columns are NOT NULL; uniqueness on business keys where applicable.
- **Dimensional sanity:** Fact tables have clear grain; all dimensional FKs present; measures are numeric; dates typed correctly.
- **Normalization & indexes:** Simple heuristic score (e.g., no mixed grains; indexes on FK columns; no free-text IDs as PKs).
- **Test inserts:** Optional synthetic row inserts to confirm constraints.
    
**Human-in-the-loop**
- Short rubric (1–5) on: _fit to business question, grain correctness, reuse of conformed dimensionss, extensibility, clarity of assumptions_.
- Compare against a **baseline template library** (star schemas for common BI domains) and report diffs.

**Metrics**
- Generation success rate (valid JSON + compiles), average lint score, # of review edits, time-to-first-DDL, user satisfaction, and adoption (schemas exported/PRs opened).

# Observability Plan

- **Tracing & logging:** Langchain spans for: retrieval, LLM call, validation, rendering. Correlate with request IDs.
- **Usage analytics:** Events for run start/success/failure, token counts, latency, cost, and feature flags (RAG on/off, dialect).
- **Error tracking:** Capture exceptions (pydantic validation errors, SQL compile errors), store failing payloads with redaction.
- **Feedback loop:** Store user ratings and edit diffs; surface top prompt failures and common corrections to refine templates.

---
# Why this is simple but useful

- Keeps scope tight: **NL → JSON design & rationale→ SQL DDL + lint/compile**.
- Produces artifacts BI teams actually need: reviewable SQL and clear assumptions.
