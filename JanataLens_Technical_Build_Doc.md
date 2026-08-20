# JanataLens — Technical Design & Build Document
### Patchamomma Build Phase Submission (Google Cloud)

---

## 1. Problem & Scope Statement

**Problem:** Citizens cannot see, in one place, whether their elected representative's allocated funds turned into real, verifiable, on-ground outcomes, cross-referenced with how the justice system is performing in their area. Government data (PRS, MPLADS/eSAKSHI, NJDG, CAG) exists but is fragmented, inconsistently formatted, and not designed for a first-time citizen user.

**Build Phase Scope (realistic for a hackathon-length build):** A single-state, single-feature-pair MVP:
- Representative track record (attendance, questions, bills) — structured data, low ingestion complexity.
- Fund allocation dashboard (MPLADS/state scheme) with a clearly labeled "verification confidence" score, not a false claim of ground-truth verification.

Judiciary integration and crowd-verification are Phase 2/3 — **explicitly out of scope for the Build Phase demo**, documented here as roadmap so judges see the full vision without you overcommitting on the demo.

---

## 2. System Architecture (Google Cloud)

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES (external)                   │
│  data.gov.in (MPLADS) │ PRS/Sansad │ state assembly portals       │
└──────────────┬────────────────────┬───────────────────┬──────────┘
               │                    │                   │
               ▼                    ▼                   ▼
    ┌─────────────────────────────────────────────────────────┐
    │  INGESTION LAYER                                          │
    │  Cloud Scheduler → Cloud Run job (Python) → raw fetch      │
    │  (API pulls where available; PDF/HTML scrape otherwise)    │
    └───────────────────────┬───────────────────────────────────┘
                            ▼
    ┌─────────────────────────────────────────────────────────┐
    │  BRONZE — Cloud Storage (GCS)                             │
    │  raw/{source}/{yyyy-mm-dd}/{file}                          │
    │  stores raw JSON/HTML/PDF + ingestion metadata             │
    └───────────────────────┬───────────────────────────────────┘
                            ▼
    ┌─────────────────────────────────────────────────────────┐
    │  TRANSFORM — Cloud Run (batch) / Cloud Functions           │
    │  - PDF/table extraction (Document AI or pdfplumber)        │
    │  - Vertex AI (Gemini) for unstructured text → structured   │
    │    fields, ALWAYS flagged "LLM-extracted, unverified"      │
    │    until a human review step clears it                     │
    │  - Entity resolution against canonical master (below)      │
    └───────────────────────┬───────────────────────────────────┘
                            ▼
    ┌─────────────────────────────────────────────────────────┐
    │  SILVER/GOLD — BigQuery                                   │
    │  Cleaned, deduplicated, entity-resolved fact + dim tables   │
    └───────────────────────┬───────────────────────────────────┘
                            ▼
    ┌─────────────────────────────────────────────────────────┐
    │  API LAYER — Cloud Run (FastAPI) or Cloud Functions        │
    │  REST endpoints serving dashboard queries from BigQuery    │
    └───────────────────────┬───────────────────────────────────┘
                            ▼
    ┌─────────────────────────────────────────────────────────┐
    │  FRONTEND — Firebase Hosting + React                       │
    │  Citizen view (simple) + Data view (tables, citations)     │
    └─────────────────────────────────────────────────────────┘

  Cross-cutting: Cloud IAM (least-privilege service accounts),
  Cloud Logging (pipeline observability), Secret Manager (any API keys)
```

**Why this maps well to GCP for a hackathon build:**
- **Cloud Run** for both ingestion jobs and the API layer keeps everything serverless and cheap — pay-per-invocation, no idle cost between demo runs.
- **BigQuery** is the right fit for gold-layer analytical queries (aggregations by constituency/district/state) and has a generous free tier for a hackathon-scale dataset.
- **Vertex AI (Gemini)** for PDF/report extraction is the most demo-worthy "AI" component of the project, but must be scoped honestly — see Section 6 on why this is flagged, not trusted blindly.
- **Firebase Hosting** is the fastest path to a deployed, shareable frontend URL for judges.

---

## 3. Data Model (BigQuery Schema)

### 3.1 Canonical Entity Tables (build this first — everything depends on it)

```sql
-- dim_representative
CREATE TABLE dim_representative (
  rep_id STRING NOT NULL,          -- canonical ID you assign, e.g. "MP-UP-VARANASI-2024"
  full_name STRING NOT NULL,
  house STRING,                    -- 'Lok Sabha' | 'Rajya Sabha' | 'State Assembly'
  party STRING,
  constituency_id STRING,
  term_start DATE,
  term_end DATE,
  source_url STRING,               -- where this record was sourced from
  last_updated TIMESTAMP
);

-- dim_constituency
CREATE TABLE dim_constituency (
  constituency_id STRING NOT NULL,
  constituency_name STRING NOT NULL,
  district_id STRING,
  state_id STRING,
  constituency_type STRING         -- 'Lok Sabha' | 'Assembly'
);

-- dim_district / dim_state — straightforward lookup tables
```

### 3.2 Fact Tables

```sql
-- fact_attendance_activity (from PRS/Sansad)
CREATE TABLE fact_attendance_activity (
  rep_id STRING,
  session_id STRING,
  attendance_pct FLOAT64,
  questions_asked INT64,
  bills_sponsored INT64,
  debates_participated INT64,
  source_url STRING,
  source_tier STRING,              -- 'T1_OFFICIAL'
  ingested_at TIMESTAMP
);

-- fact_fund_allocation (from MPLADS/state scheme portals)
CREATE TABLE fact_fund_allocation (
  rep_id STRING,
  constituency_id STRING,
  fiscal_year STRING,
  amount_sanctioned_inr NUMERIC,
  amount_released_inr NUMERIC,
  amount_utilized_inr NUMERIC,
  work_id STRING,
  work_description STRING,
  work_status STRING,              -- 'sanctioned' | 'in_progress' | 'completed' (per official source)
  implementing_agency STRING,
  source_url STRING,
  source_tier STRING,
  ingested_at TIMESTAMP
);

-- fact_verification (Phase 3 — schema only, not populated in Build Phase demo)
CREATE TABLE fact_verification (
  work_id STRING,
  verification_type STRING,        -- 'citizen_report' | 'rti_response' | 'cag_audit'
  verification_confidence STRING,  -- 'T3_UNVERIFIED' | 'T2_VERIFIED_SECONDARY'
  submitted_by STRING,
  evidence_url STRING,
  moderation_status STRING,
  created_at TIMESTAMP
);
```

**Key design decision:** `source_tier` is a column on every fact row, not a UI afterthought. Every query and every dashboard card must be able to answer "what tier is this number" — this is your credibility mechanism, and it needs to be structural, not cosmetic.

---

## 4. Entity Resolution Approach (the hardest technical problem)

Representative and constituency names are inconsistent across sources (spelling variants, delimitation changes, mid-term party switches). For the Build Phase:

1. **Manually curate** the canonical `dim_representative` and `dim_constituency` tables for your one pilot state — do not attempt automated entity resolution across the whole country in the time you have.
2. Write a **fuzzy-matching reconciliation script** (Python, `rapidfuzz` or similar) that maps incoming source records to canonical IDs, with any match below a confidence threshold routed to a manual review CSV rather than auto-inserted.
3. Log every unresolved/ambiguous match — this becomes a visible "data quality backlog" you can show judges as evidence of rigor, not a hidden gap.

This is explicitly the piece of the system most aligned with the kind of governed, medallion-style data engineering discipline worth highlighting in your submission write-up — it's the unglamorous part that makes the rest of the product trustworthy.

---

## 5. API Design (Cloud Run / FastAPI)

```
GET /api/v1/representative/{rep_id}
  → { name, party, house, constituency, attendance_summary, fund_summary }

GET /api/v1/constituency/{constituency_id}/funds?fiscal_year=2024-25
  → [ { work_id, description, amount_sanctioned, amount_utilized,
        status, source_tier, source_url } ]

GET /api/v1/search?pincode={pincode}
  → resolves pincode → constituency_id → redirects to constituency view
  (Build Phase: hardcode a pincode-to-constituency lookup table for the
   pilot state; a full national pincode mapping is out of scope)

GET /api/v1/representative/{rep_id}/citations
  → returns every source_url backing the displayed numbers —
    this endpoint is your credibility feature made concrete
```

Keep every response payload carrying `source_tier` and `source_url` per field — the frontend should never display a number without being able to show where it came from on click.

---

## 6. Responsible-AI / Data Integrity Guardrails (important for judging criteria)

Google Cloud hackathons typically score on responsible AI use and real-world credibility, not just working code. Bake these in and call them out explicitly in your submission:

- **LLM-extracted fields are never auto-published.** Any figure pulled from a PDF via Gemini/Document AI is written to a `pending_review` table, not the gold fact table, until a human confirms it against the source document. State this constraint clearly in your demo narrative — it's a strength, not a limitation.
- **No editorializing.** The system states what official sources say; it does not rate, score, or characterize a representative's performance. Aggregation and citation only.
- **No claims about unverified on-ground impact.** Section 3.2's `fact_verification` table is schema-only in the Build Phase specifically so you don't accidentally ship a feature that implies confirmed ground-truth you haven't actually verified.
- **Data lineage on every number.** `source_url` + `ingested_at` on every fact row, surfaced via the `/citations` endpoint — this is both a technical and legal safeguard.

---

## 7. Build Phase Deliverable Checklist

- [ ] One pilot state chosen, data sources manually verified to exist in usable form
- [ ] `dim_representative` / `dim_constituency` canonical tables populated (can be manually curated for MVP scale — a few dozen to a few hundred rows is fine)
- [ ] Ingestion pipeline (Cloud Run job) pulling attendance/bills data (PRS-style source) into GCS bronze
- [ ] Ingestion pipeline for MPLADS/state fund data into GCS bronze
- [ ] Transform job writing cleaned records into BigQuery silver/gold tables, with `source_tier` populated
- [ ] FastAPI service on Cloud Run exposing the endpoints in Section 5
- [ ] React frontend on Firebase Hosting: constituency search → representative profile → fund table, every number clickable to its source
- [ ] A visible "Data Quality" or "About This Data" page in the frontend explaining the source-tier system — judges reward transparency about limitations
- [ ] Short demo script: pincode → constituency → representative profile → fund breakdown → click a number → see source citation

---

## 8. Roadmap Beyond Build Phase (for the pitch, not the demo)

- **Phase 2:** NJDG judiciary integration (district pendency/disposal/vacancy) — same entity-resolution pattern extended to court jurisdictions.
- **Phase 3:** Moderated Tier-3 citizen verification queue (populate `fact_verification`), RTI-response ingestion pipeline.
- **Phase 4:** National scale-out, API/data-licensing tier for journalists and researchers, manifesto promise-tracking (editorial layer requiring a fact-checking process, not pure pipeline work).

---

## 9. Legal/Regulatory Notes to Mention in Submission Write-Up

- All figures are sourced-and-cited from official government publications (data.gov.in, MPLADS/eSAKSHI, PRS) — the system aggregates and cites, it does not allege.
- DPDP Act, 2023 considerations apply once citizen-submitted verification (Phase 3) is built — out of scope for Build Phase, but worth a one-line mention that you're aware of it.
- No judgment-quality commentary anywhere in the system, now or in the roadmap — structural/statistical transparency only.

---

*This document is scoped specifically for a time-boxed hackathon Build Phase: one state, two data domains, a working deployed demo on GCP, and an honest roadmap for the rest. Verify current API/portal access and rate limits for your chosen pilot state's data sources before finalizing ingestion code, as government portal structures can change without notice.*
