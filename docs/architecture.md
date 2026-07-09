# Architecture: AI-Powered Restaurant Recommendation System

> Derived from [`context.md`](context.md) and the Zomato use-case problem statement.  
> This document describes **what** to build, **how** components interact, and **why** key design choices are made.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Design Principles](#2-design-principles)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Component Architecture](#4-component-architecture)
5. [Data Architecture](#5-data-architecture)
6. [Request Lifecycle](#6-request-lifecycle)
7. [Integration Layer & LLM Design](#7-integration-layer--llm-design)
8. [Application Layers](#8-application-layers)
9. [Interface & Output Contract](#9-interface--output-contract)
10. [Cross-Cutting Concerns](#10-cross-cutting-concerns)
11. [Technology Decisions](#11-technology-decisions)
12. [Deployment Topology](#12-deployment-topology)
13. [Future Extensions](#13-future-extensions)

---

## 1. Executive Summary

The system is a **preference-driven restaurant recommender** built as a full-stack web application. It:

1. Ingests and normalizes the Zomato restaurant dataset (~51k rows) from Hugging Face, persisting it as a local Parquet cache.
2. Accepts structured user preferences (location, budget, cuisine, minimum rating) through a React frontend.
3. **Deterministically filters** the cached dataset to a bounded candidate pool via the FastAPI backend.
4. Uses a **Groq-hosted LLM (LLaMA-3) only on that candidate pool** to rank, explain, and optionally summarize results.
5. Returns a structured JSON response consumed by the frontend to display ranked recommendation cards.

The architecture deliberately separates **factual retrieval** (Parquet cache + deterministic filters) from **subjective reasoning** (LLM), keeping recommendations traceable to real dataset records and minimizing hallucination risk.

---

## 2. Design Principles

| Principle | Implication |
|-----------|-------------|
| **Grounded recommendations** | Every suggested restaurant must exist in the filtered candidate list passed to the LLM. |
| **Filter first, reason second** | Hard constraints (location, min rating, budget band) are applied before the LLM sees data. |
| **Bounded LLM context** | Send only top-N candidates (e.g., 15–30) to control cost, latency, and token limits. |
| **Structured in, structured out** | User input and LLM output should use schemas (JSON) where possible for validation. |
| **Explainability by default** | Each result includes an AI-generated “why this fits” tied to stated preferences. |
| **Layered interfaces** | Core recommendation modules are UI-agnostic; the same pipeline powers the React frontend, the Streamlit app, and the CLI entry point. |

---

## 3. High-Level Architecture

### 3.1 Logical View

```mermaid
flowchart TB
    subgraph Client["Presentation Layer"]
        UI["React Frontend (Vercel)<br/>Streamlit App / CLI"]
    end

    subgraph App["Application Layer"]
        API[Preference Controller]
        ORCH[Recommendation Orchestrator]
    end

    subgraph Core["Domain / Core Services"]
        FILTER[Candidate Filter Service]
        PROMPT[Prompt Builder]
        PARSER[LLM Response Parser]
    end

    subgraph Data["Data Layer"]
        REPO[Restaurant Repository]
        CACHE[(In-Memory / Disk Cache)]
        HF[(Hugging Face Dataset)]
    end

    subgraph External["External Services"]
        LLM[LLM Provider API]
    end

    UI --> API
    API --> ORCH
    ORCH --> FILTER
    FILTER --> REPO
    REPO --> CACHE
    CACHE --> HF
    ORCH --> PROMPT
    PROMPT --> LLM
    LLM --> PARSER
    PARSER --> ORCH
    ORCH --> UI
```

### 3.2 Layered Responsibility Model

| Layer | Components | Responsibility |
|-------|------------|----------------|
| **Presentation** | UI, forms, result cards | Collect preferences; display ranked recommendations |
| **Application** | Orchestrator, controllers | Coordinate filter → prompt → LLM → parse → format |
| **Domain** | Filter rules, budget mapping, ranking policy | Business logic independent of UI and LLM vendor |
| **Data** | Loader, preprocessor, repository | Ingest HF dataset; serve queryable restaurant records |
| **Integration** | Prompt templates, LLM client | Vendor-specific API calls; retries and timeouts |
| **External** | Hugging Face, LLM API | Source data and generative reasoning |

---

## 4. Component Architecture

### 4.1 Component Map

```
┌─────────────────────────────────────────────────────────────────┐
│                     Presentation Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Preference   │  │ Results      │  │ Error / Empty State  │  │
│  │ Form         │  │ Renderer     │  │ Handler              │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                   Recommendation Orchestrator                   │
│   validate prefs → filter → build prompt → call LLM → merge     │
└─────┬──────────────────┬──────────────────┬─────────────────────┘
      │                  │                  │
┌─────▼─────┐    ┌───────▼───────┐   ┌──────▼──────┐
│ Preference│    │ Candidate     │   │ LLM         │
│ Validator │    │ Filter        │   │ Integration │
└───────────┘    └───────┬───────┘   └──────┬──────┘
                         │                  │
                  ┌──────▼───────┐   ┌──────▼──────┐
                  │ Restaurant   │   │ Prompt      │
                  │ Repository   │   │ Builder +   │
                  │              │   │ Parser      │
                  └──────┬───────┘   └─────────────┘
                         │
                  ┌──────▼───────┐
                  │ Data Loader  │
                  │ + Preprocess │
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │ Hugging Face │
                  │ Dataset      │
                  └──────────────┘
```

### 4.2 Component Specifications

#### 4.2.1 Data Loader & Preprocessor

**Purpose:** One-time (or scheduled) ingestion from Hugging Face into an application-ready store.

| Concern | Design |
|---------|--------|
| **Source** | `ManikaSaini/zomato-restaurant-recommendation` on Hugging Face |
| **Load strategy** | Download via `datasets` library; persist as a Parquet file for fast in-memory reload at startup |
| **Normalization** | Trim strings; standardize city names; parse ratings as float; map cost to numeric + budget tier |
| **Schema mapping** | Map raw columns → canonical `Restaurant` model (see §5) |
| **Validation** | Drop or flag rows missing name, location, or rating; log counts |

**Outputs:** Clean `Restaurant[]` collection indexed for filtering (by city, cuisine, rating, cost).

---

#### 4.2.2 Restaurant Repository

**Purpose:** Abstract read access to restaurant data for the filter service.

| Operation | Description |
|-----------|-------------|
| `get_all()` | Return the full normalized restaurant collection for use by the filter service |
| `distinct_cities()` | Return a sorted list of distinct city names, used to populate the location dropdown |
| `load()` / `ensure_loaded()` | Load the Parquet cache into memory on startup or on first access |

**Implementation:** In-memory store backed by the Parquet cache file. The `CandidateFilterService` retrieves the full collection via `get_all()` and applies all filtering in Python — the repository is a read-only data source, not a query engine.

---

#### 4.2.3 Preference Validator

**Purpose:** Validate and normalize user input before filtering.

| Field | Validation | Normalization |
|-------|------------|---------------|
| `location` | Required; non-empty string | Title-case; alias map (e.g., "Bengaluru" → "Bangalore") |
| `budget` | Enum: `low` \| `medium` \| `high` | Map to cost ranges (config-driven) |
| `cuisine` | Optional; string or list | Lowercase; partial match token |
| `min_rating` | Optional; 0.0–5.0 | Default e.g. 3.5 if omitted |
| `additional_preferences` | Optional free text | Passed through to LLM only (soft signal) |

---

#### 4.2.4 Candidate Filter Service

**Purpose:** Apply **hard filters** deterministically; produce a bounded candidate list.

**Filter pipeline (ordered):**

1. **Location** — exact or fuzzy match on city/area field.
2. **Minimum rating** — `rating >= min_rating`.
3. **Cuisine** — contains match on cuisine string (supports multi-cuisine labels).
4. **Budget** — map `low|medium|high` to `cost_for_two` or `price_range` bands.
5. **Cap** — if results > `MAX_CANDIDATES` (e.g., 25), sort by rating desc and truncate.

**Empty result handling:** Return structured empty state with suggestions (relax rating, broaden location, change budget).

---

#### 4.2.5 Prompt Builder

**Purpose:** Construct a system + user prompt that constrains the LLM to reason only over provided JSON candidates.

**Prompt structure:**

- **System:** Role (restaurant advisor), rules (only recommend from list, output JSON schema, cite preference alignment).
- **User:** Serialized `UserPreferences` + `CandidateRestaurant[]` (id, name, cuisine, rating, cost, location snippet).
- **Constraints:** Max recommendations (e.g., top 5), require `restaurant_id`, `rank`, `explanation`; optional `summary`.

---

#### 4.2.6 LLM Integration Client

**Purpose:** Call the LLM provider with retries, timeouts, and response extraction.

| Concern | Approach |
|---------|----------|
| **Provider** | Groq API (LLaMA-3 model family) |
| **Temperature** | Low (0.2) for deterministic, consistent ranking |
| **Output format** | Structured output enforced via system prompt instructions and post-response validation — not a provider JSON mode |
| **Retry** | One automatic retry on transient failure before escalating to the fallback path |
| **Fallback** | If LLM fails or returns unparseable output: deterministic top-N ranked by rating with template explanations |

---

#### 4.2.7 LLM Response Parser & Merger

**Purpose:** Parses LLM JSON; validates IDs exist in candidates; merges with full restaurant records.

**Validation rules:**

- Strip markdown code fences before attempting JSON parse.
- Reject any `restaurant_id` not present in the candidate set — unknown IDs trigger a parse error.
- Reject duplicate or out-of-bounds rank values.
- If the full parse fails, the orchestrator activates the fallback path; there is no partial backfill.

---

#### 4.2.8 Recommendation Orchestrator

**Purpose:** Single entry point for the “recommend” use case.

```python
# Pseudocode contract
def recommend(preferences: UserPreferences) -> RecommendationResponse:
    filter_result = filter_service.apply(preferences)
    if not filter_result.candidates:
        return empty_response(suggestions=filter_result.suggestions)
    try:
        llm_raw = llm_client.complete(preferences, filter_result.candidates)
        response = parser.parse_llm_response(llm_raw, filter_result.candidates)
        fallback_used = False
    except Exception:
        response = fallback_ranking(filter_result.candidates)
        fallback_used = True
    return response.with_metadata(filter_result, fallback_used)
```

---

#### 4.2.9 Results Formatter / Presentation

**Purpose:** Map domain objects to UI-ready cards.

Each **RecommendationCard** includes:

- Restaurant name  
- Cuisine  
- Rating  
- Estimated cost  
- AI-generated explanation  
- Optional: overall summary paragraph  

---

## 5. Data Architecture

### 5.1 Canonical Domain Models

#### Restaurant (normalized)

```json
{
  "id": "string",
  "name": "string",
  "location": "string",
  "city": "string",
  "cuisine": "string",
  "rating": 4.2,
  "cost_for_two": 800,
  "budget_tier": "medium",
  "raw": {}
}
```

#### UserPreferences

```json
{
  "location": "Bangalore",
  "budget": "medium",
  "cuisine": "Italian",
  "min_rating": 4.0,
  "additional_preferences": "family-friendly, quick service"
}
```

#### Recommendation (output item)

The `Recommendation` object returned by the LLM contains only the fields needed to re-associate with the candidate pool — restaurant data fields are not duplicated.

```json
{
  "restaurant_id": "abc123",
  "rank": 1,
  "explanation": "Matches your Italian preference and medium budget..."
}
```

#### RecommendationResponse

All metadata fields are top-level — there is no nested `metadata` wrapper.

```json
{
  "recommendations": [],
  "summary": "Optional overview of the shortlist.",
  "filters_applied": ["location", "min_rating", "cuisine", "budget"],
  "candidates_considered": 18,
  "suggestions": [],
  "fallback_used": false,
  "model_version": null
}
```

### 5.2 Budget Tier Mapping (configurable)

| Tier | Typical `cost_for_two` range (INR) | Notes |
|------|-------------------------------------|-------|
| `low` | 0 – 500 | Adjust after inspecting dataset distribution |
| `medium` | 501 – 1500 | Percentile-based thresholds preferred |
| `high` | 1500+ | |

*Calibrate tiers using dataset histograms during preprocessing.*

### 5.3 Data Flow Diagram

```mermaid
sequenceDiagram
    participant HF as Hugging Face
    participant DL as Data Loader
    participant PP as Preprocessor
    participant DB as Repository
    participant FS as Filter Service
    participant PB as Prompt Builder
    participant LLM as LLM API
    participant OR as Orchestrator

    HF->>DL: Download dataset
    DL->>PP: Raw records
    PP->>DB: Normalized Restaurant[]
    Note over DB: Startup / refresh

    OR->>FS: UserPreferences
    FS->>DB: get_all()
    DB-->>FS: Full restaurant collection
    FS->>FS: Apply filters in-memory
    FS-->>OR: Bounded candidates (capped at MAX_CANDIDATES)
    OR->>PB: Prefs + candidates
    PB->>LLM: Structured prompt
    LLM-->>OR: Ranked JSON + explanations
    OR-->>OR: Merge + validate IDs
```

---

## 6. Request Lifecycle

End-to-end flow for a single recommendation request:

```mermaid
flowchart LR
    A[User submits preferences] --> B[Validate & normalize]
    B --> C{Valid?}
    C -->|No| D[Return validation errors]
    C -->|Yes| E[Filter repository]
    E --> F{Any candidates?}
    F -->|No| G[Empty state + suggestions]
    F -->|Yes| H[Build LLM prompt]
    H --> I[LLM rank + explain]
    I --> J{Parse OK?}
    J -->|No| K[Fallback: rating sort + template text]
    J -->|Yes| L[Merge with restaurant records]
    K --> M[Format response]
    L --> M
    M --> N[Display to user]
```

### 6.1 Latency Budget (target)

| Stage | Target |
|-------|--------|
| Filter (in-memory) | < 100 ms |
| LLM call | 2–8 s (depends on model) |
| Parse + format | < 50 ms |
| **Total perceived** | < 10 s with loading indicator |

---

## 7. Integration Layer & LLM Design

### 7.1 Why a Dedicated Integration Layer

The integration layer sits between **filtered structured data** and the **LLM**, ensuring:

- Prompts are versioned and testable.
- Candidate sets are serialized consistently.
- The LLM cannot invent restaurants outside the provided list (enforced in prompt + post-validation).

### 7.2 Prompt Design Guidelines

**System instructions (summary):**

1. You are a restaurant recommendation assistant for Indian cities (Zomato-style).
2. Recommend **only** from the `candidates` array.
3. Rank by fit to `user_preferences`; use `additional_preferences` as soft signals.
4. Return **valid JSON** matching the output schema.
5. Each `explanation` must reference specific preference fields.

**Anti-hallucination measures:**

- Include explicit `restaurant_id` for each candidate in the serialized prompt payload.
- Post-parse: reject any recommendation whose `restaurant_id` is not in the candidate set.
- Prefer JSON schema / function calling when the provider supports it.

### 7.3 Example Prompt Skeleton

```
SYSTEM:
You recommend restaurants only from the provided candidate list.
Output JSON: { "recommendations": [...], "summary": "..." }

USER:
Preferences:
{ "location": "Delhi", "budget": "low", "cuisine": "Chinese", "min_rating": 4.0, "additional_preferences": "quick service" }

Candidates (do not invent others):
[
  { "restaurant_id": "abc1", "name": "...", "cuisine": "...", "rating": 4.2, "cost_for_two": 400, "location": "Connaught Place, Delhi" },
  ...
]

Return top 5 ranked recommendations with explanations.
```

### 7.4 Fallback Strategy

| Failure | Behavior |
|---------|----------|
| LLM timeout or API error | Retry once; if retry fails, return top-N by rating with template explanations |
| Invalid or unparseable JSON | Immediate fallback — no partial backfill |
| Unknown `restaurant_id` in response | Parse error triggers full fallback path |
| Empty filter results | Return empty response with actionable suggestions; no LLM call is made |

---

## 8. Application Layers

### 8.1 Module Structure

```
zomato-ai/
├── app/
│   ├── api.py                  # FastAPI routes, CORS configuration, repository lifecycle
│   ├── main.py                 # CLI entry point
│   └── streamlit_app.py        # Streamlit interface (secondary UI)
├── core/
│   ├── models.py               # Pydantic domain models: Restaurant, UserPreferences, Recommendation
│   ├── orchestrator.py         # recommend() workflow coordinator
│   ├── filter.py               # Candidate filter service
│   ├── validator.py            # Preference input validation
│   └── formatter.py            # Output formatting
├── data/
│   ├── loader.py               # Hugging Face dataset download + Parquet cache builder
│   ├── preprocessor.py         # Normalization: fields, budget tiers, city aliases
│   ├── repository.py           # In-memory restaurant store
│   └── cache/
│       ├── restaurants.parquet # Pre-built data cache
│       └── cache_metadata.json # Ingest stats and budget thresholds
├── llm/
│   ├── client.py               # Groq API client with retry logic
│   ├── prompts.py              # Structured prompt builder
│   └── parser.py               # LLM JSON response parser and fallback generator
├── config/
│   └── settings.py             # Pydantic-settings: budget bands, MAX_CANDIDATES, API keys
├── frontend/                   # React + Vite + Tailwind CSS (deployed on Vercel)
│   └── src/
│       └── components/         # Hero, PreferencesForm, Recommendations
└── tests/                      # pytest unit and integration tests
```

### 8.2 API Surface

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/recommend` | Body: `UserPreferences` → `RecommendationResponse` |
| `GET` | `/api/cities` | Returns sorted distinct city names for the location dropdown |
| `GET` | `/api/health` | Liveness check |
| `GET` | `/docs` | Auto-generated interactive API documentation (Swagger UI) |

---

## 9. Interface & Output Contract

### 9.1 Input Form (Presentation)

| Field | Control type | Required |
|-------|--------------|----------|
| Location | Dropdown populated from `/api/cities` | Yes |
| Budget | Radio buttons: Low / Medium / High | Yes |
| Cuisine | Text input (comma-separated) | No |
| Minimum rating | Slider 0.0–5.0 (step 0.1) | No |

### 9.2 Output Card Layout

Each recommendation card displays rank, a computed match score, the restaurant identifier, and the AI-generated explanation.

```
┌─────────────────────────────────────────────┐
│  Rank  #1                      Match  95%   │
│                                             │
│  Restaurant Name                            │
│                                             │
│  AI Reasoning                               │
│  "Great fit for your medium budget and      │
│   Italian cuisine preference..."            │
│                                             │
│  → AI Ranked  (revealed on hover)          │
└─────────────────────────────────────────────┘
```

An optional **Summary** paragraph (returned by the LLM) is displayed above the card grid. A **Fallback** badge is shown when the LLM path was not used and results are ranked by rating instead.

---

## 10. Cross-Cutting Concerns

### 10.1 Security

| Item | Approach |
|------|----------|
| API keys | Environment variables (`GROQ_API_KEY`, etc.); never commit |
| User input | Sanitize free-text fields; length limits on `additional_preferences` |
| LLM data | Do not log full prompts in production without redaction |

### 10.2 Observability

- Log: filter duration, candidate count, LLM latency, parse success/failure.
- Metrics: empty-result rate, fallback rate, average recommendations returned.

### 10.3 Error Handling

| Error type | User-facing message | Internal action |
|------------|---------------------|-----------------|
| Dataset load failure | “Unable to load restaurant data” | Log stack trace; retry |
| No matches | “No restaurants match; try relaxing filters” | Skip LLM |
| LLM failure | “Showing top-rated matches” | Fallback ranking |
| Invalid preferences | Field-level validation errors | 400 response |

### 10.4 Testing Strategy

| Layer | Tests |
|-------|-------|
| Preprocessor | Schema mapping, null handling, budget tier assignment |
| Filter | Location/cuisine/budget/rating combinations; edge cases |
| Parser | Valid JSON, unknown IDs, malformed LLM output |
| Orchestrator | Integration test with mocked LLM |
| E2E | Golden path: Delhi + medium + Chinese + min 4.0 |

### 10.5 Performance & Caching

- **Dataset cache:** Load once at startup; refresh on demand or daily.
- **Location/cuisine indexes:** Precompute distinct values for UI dropdowns.
- **LLM:** Limit candidates to 25; request only top 5 recommendations.

---

## 11. Technology Decisions

| Concern | Implemented | Rationale |
|---------|-------------|-----------|
| Language | Python 3.11+ | Ecosystem fit for data processing, LLM integration, and FastAPI |
| Dataset | Hugging Face `datasets` library | Direct access to `ManikaSaini/zomato-restaurant-recommendation` |
| Data format | Parquet + in-memory (Pandas) | Fast columnar reads; cache built once at deploy time |
| Backend framework | FastAPI | Native Pydantic support; auto-generated OpenAPI documentation |
| Frontend | React + Vite + Tailwind CSS | Component model, fast builds, zero-config Vercel deployment |
| LLM provider | Groq (LLaMA-3) | Low-latency inference; free tier suitable for portfolio use |
| Configuration | `pydantic-settings` | Unified env var and `.env` management with type safety |
| Frontend hosting | Vercel | Static bundle deployment; native Vite project support |
| Backend hosting | Render | Managed Python web service with configurable build and start commands |

---

## 12. Deployment Topology

### 12.1 Local / Demo (MVP)

```mermaid
flowchart LR
    DEV[Developer Machine]
    DEV --> ST[Streamlit App]
    ST --> MEM[(In-Memory Dataset)]
    ST --> LLM[Cloud LLM API]
```

- Single process; dataset loaded on start.
- Suitable for case study demos and interviews.

### 12.2 Production (Deployed)

```mermaid
flowchart TB
    USER["Browser"] --> FE["React Frontend\n(Vercel)"]
    FE -->|"POST /api/recommend\nGET /api/cities"| BE["FastAPI Backend\n(Render)"]
    BE --> CACHE["Parquet Cache\ndata/cache/"]
    CACHE -.->|"Built at deploy time\nvia data.loader"| HF["Hugging Face\nDataset"]
    BE -->|"Filtered candidates + prompt"| LLM["Groq API\n(LLaMA-3)"]
    LLM -->|"Ranked JSON"| BE
```

- **Frontend (Vercel):** React + Vite SPA. Reads `VITE_API_URL` at build time to connect to the Render backend.
- **Backend (Render):** FastAPI + Uvicorn. The Parquet cache is built during the Render build step (`data.loader --refresh`) and loaded into memory on startup.
- **LLM (Groq):** `GROQ_API_KEY` is injected as a Render environment variable. No LLM responses are cached or persisted.

---

## 13. Future Extensions

| Extension | Architectural impact |
|-----------|----------------------|
| User accounts & history | Add auth service + preference store |
| Geospatial search | Replace city string match with lat/long radius |
| Embeddings + semantic cuisine match | Vector DB for soft matching before LLM |
| A/B testing prompts | Prompt registry with version flags |
| Feedback loop | Thumbs up/down stored; fine-tune prompts or reranker |
| Multi-language explanations | Locale field in preferences → prompt language |

---

## Appendix A: Mapping to `context.md` Workflow

| Context workflow step | Architecture components |
|-----------------------|-------------------------|
| 1. Data Ingestion | Data Loader, Preprocessor, Repository |
| 2. User Input | Presentation form, Preference Validator |
| 3. Integration Layer | Filter Service, Prompt Builder |
| 4. Recommendation Engine | LLM Client, Parser, Orchestrator |
| 5. Output Display | Formatter, UI results renderer |

---

## Appendix B: Key Constraints (from context)

1. **Grounded data** — Enforced by filter-then-LLM pipeline and ID validation.
2. **LLM role** — Ranking and explanation only; not primary data source.
3. **User experience** — Zomato-style, preference-driven, readable cards.
4. **Actionable output** — Name, cuisine, rating, cost, and explanation on every item.

---

## Related Documents

- [`context.md`](context.md) — Project scope and workflow summary  
- [`docs/Problem statement.txt`](docs/Problem%20statement.txt) — Original assignment brief  
