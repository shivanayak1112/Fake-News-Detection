# Fake News Detection - System Architecture

**Project:** AI-Powered Fake News Detection & Claim Verification  
**Version:** 1.0.0  
**Last Updated:** 2026-08-18

---

## TABLE OF CONTENTS

1. [High-Level Architecture](#high-level-architecture)
2. [System Components](#system-components)
3. [Data Flow Architecture](#data-flow-architecture)
4. [Technology Stack](#technology-stack)
5. [Backend Architecture](#backend-architecture)
6. [Frontend Architecture](#frontend-architecture)
7. [Service Dependencies](#service-dependencies)
8. [Database Schema](#database-schema)
9. [API Endpoints Map](#api-endpoints-map)
10. [Deployment Architecture](#deployment-architecture)

---

## HIGH-LEVEL ARCHITECTURE

### 3-Layer Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      PRESENTATION LAYER                         │
│                      (React Frontend)                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  React SPA (Vite)                                          │ │
│  │  ├─ Dashboard Page (/)          - Model metrics display    │ │
│  │  ├─ Analyze Page (/analyze)     - Article input & results  │ │
│  │  ├─ Router (React Router v7)    - Navigation              │ │
│  │  └─ API Client (axios)          - HTTP communication       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                      HTTP/JSON ↔ REST API                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓↑
         ┌────────────────────────────────────────┐
         │    NETWORK LAYER                       │
         │ http://127.0.0.1:8000                  │
         │ CORS Headers | Request/Response        │
         └────────────────────────────────────────┘
                              ↓↑
┌─────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                             │
│                   (FastAPI Backend)                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  FastAPI + Uvicorn (Async ASGI)                            │ │
│  │  ├─ Routes Layer                                           │ │
│  │  │  ├─ routes_analysis.py      - /api/analyze endpoints   │ │
│  │  │  ├─ routes_metrics.py       - /api/metrics endpoint    │ │
│  │  │  ├─ routes_health.py        - /api/health endpoint     │ │
│  │  │  ├─ routes_history.py       - /api/history endpoints   │ │
│  │  │  └─ routes_feedback.py      - /api/feedback endpoint   │ │
│  │  │                                                          │ │
│  │  ├─ Services Layer (Business Logic)                        │ │
│  │  │  ├─ ml_service.py                  - ML inference      │ │
│  │  │  ├─ claim_extractor.py             - Claim extraction  │ │
│  │  │  ├─ evidence_retriever.py          - Evidence search   │ │
│  │  │  ├─ claim_verification_service.py  - Verify claims     │ │
│  │  │  ├─ decision_engine.py             - Hybrid decision   │ │
│  │  │  ├─ explanation_service.py         - Explain results   │ │
│  │  │  ├─ article_url_service.py         - URL extraction    │ │
│  │  │  └─ source_scoring_service.py      - Score sources    │ │
│  │  │                                                          │ │
│  │  ├─ Data Layer                                             │ │
│  │  │  ├─ models/schemas.py      - Pydantic data models      │ │
│  │  │  ├─ database/connection.py - MongoDB manager           │ │
│  │  │  └─ database/repositories.py - CRUD operations         │ │
│  │  │                                                          │ │
│  │  ├─ Providers Layer (External APIs)                        │ │
│  │  │  ├─ base_provider.py            - Provider interface   │ │
│  │  │  ├─ google_factcheck.py         - Google API           │ │
│  │  │  ├─ web_search.py               - Web search APIs      │ │
│  │  │  └─ wikipedia_provider.py       - Wikipedia API        │ │
│  │  │                                                          │ │
│  │  ├─ Middleware                                             │ │
│  │  │  └─ CORS Middleware             - Frontend integration │ │
│  │  │                                                          │ │
│  │  └─ Utils                                                   │ │
│  │     └─ logger.py                   - Logging              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                          ↓↑ (Multiple)
        ┌─────────────────────────────────┐
        │   EXTERNAL SERVICES             │
        ├─────────────────────────────────┤
        │ • Google Fact Check API         │
        │ • Tavily Search API             │
        │ • SerpAPI                       │
        │ • Bing Search API               │
        │ • Wikipedia                     │
        └─────────────────────────────────┘
                          ↓↑
┌─────────────────────────────────────────────────────────────────┐
│                   DATA LAYER                                    │
│                  (File System & Database)                       │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  File System (Joblib Models)                               │ │
│  │  └─ outputs/                                               │ │
│  │     ├─ pipeline.joblib       - Trained ML model            │ │
│  │     ├─ vectorizer.joblib     - TF-IDF vectorizer           │ │
│  │     ├─ metrics.json          - Model performance           │ │
│  │     └─ artifact_environment.json - Model metadata          │ │
│  ├────────────────────────────────────────────────────────────┤ │
│  │  MongoDB (Optional Persistence)                            │ │
│  │  ├─ Collections:                                           │ │
│  │  │  ├─ analyses         - Complete analysis results       │ │
│  │  │  ├─ claims           - Extracted claims                │ │
│  │  │  ├─ evidence         - Retrieved evidence items        │ │
│  │  │  └─ feedback         - User feedback                   │ │
│  │  └─ Indexes:                                              │ │
│  │     ├─ (user_id)                                          │ │
│  │     ├─ (created_at)                                       │ │
│  │     └─ (final_decision)                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## SYSTEM COMPONENTS

### Component Interaction Matrix

```
┌──────────────────────────────────────────────────────────────────────────┐
│ FRONTEND (React)                                                         │
│ ┌────────────┐     ┌──────────────┐     ┌─────────────────┐            │
│ │ Dashboard  │────▶│ API Client   │◀────│ Analyze Page    │            │
│ └────────────┘     │ (axios)      │     └─────────────────┘            │
│                    └──────────────┘                                      │
└──────────────────────────────────────────────────────────────────────────┘
                            │
                    HTTP JSON (Port 5173)
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌──────▼──────────┐  ┌────▼──────────┐
│   /api/metrics │  │ /api/analyze    │  │ /api/history  │
│                │  │ /api/analyze/url│  │ /api/health   │
└────────────────┘  └─────────────────┘  └───────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
┌──────────────────────────▼────────────────────────────────────────────────┐
│ BACKEND (FastAPI)                                                        │
│                                                                          │
│ ┌─ REQUEST ROUTING ─────────────────────────────────────────────────┐  │
│ │  FastAPI Router → Route Handlers → Pydantic Validation            │  │
│ └───────────────────────────────────────────────────────────────────┘  │
│                                 │                                        │
│ ┌─ SERVICE ORCHESTRATION ──────▼──────────────────────────────────┐  │
│ │                                                                  │  │
│ │  ┌─────────────────────────────────────────────────────────┐   │  │
│ │  │ Analysis Flow:                                          │   │  │
│ │  │                                                         │   │  │
│ │  │  1. Extract article (if URL)                           │   │  │
│ │  │     └─▶ article_url_service                            │   │  │
│ │  │                                                         │   │  │
│ │  │  2. [PARALLEL PROCESSING]                              │   │  │
│ │  │     ├─▶ ML Service (TF-IDF + LogReg)                  │   │  │
│ │  │     ├─▶ Claim Extractor                                │   │  │
│ │  │     └─▶ Evidence Retriever (multi-source)              │   │  │
│ │  │                                                         │   │  │
│ │  │  3. Claim Verification                                 │   │  │
│ │  │     └─▶ claim_verification_service                     │   │  │
│ │  │                                                         │   │  │
│ │  │  4. Decision Engine                                    │   │  │
│ │  │     └─▶ Combine ML (35%) + Evidence (65%)             │   │  │
│ │  │                                                         │   │  │
│ │  │  5. Explanation Generation                             │   │  │
│ │  │     └─▶ explanation_service                            │   │  │
│ │  │                                                         │   │  │
│ │  │  6. Persistence (if DB available)                      │   │  │
│ │  │     └─▶ MongoDB repositories                           │   │  │
│ │  └─────────────────────────────────────────────────────────┘   │  │
│ └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│ ┌─ EXTERNAL API CALLS ──────────────────────────────────────────────┐  │
│ │                                                                    │  │
│ │  Evidence Providers (Concurrent):                                 │  │
│ │  ├─ Google Factcheck API ────┐                                   │  │
│ │  ├─ Tavily Web Search ───────┼─▶ evidence_retriever              │  │
│ │  ├─ SerpAPI ─────────────────┤                                   │  │
│ │  ├─ Bing Search API ─────────┤                                   │  │
│ │  └─ Wikipedia ───────────────┘                                   │  │
│ └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│ ┌─ DATA PERSISTENCE ────────────────────────────────────────────────┐  │
│ │  Motor (Async MongoDB) ◀─ Optional                                │  │
│ │  ├─ analyses (store full results)                                 │  │
│ │  ├─ claims (store extracted claims)                               │  │
│ │  └─ evidence (store evidence items)                               │  │
│ └───────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## DATA FLOW ARCHITECTURE

### Complete Request/Response Flow

```
USER SUBMITS ARTICLE IN BROWSER
│
├─▶ Frontend (React)
│   ├─ User inputs: headline + article text (or URL)
│   └─ POST http://127.0.0.1:8000/api/analyze
│       ├─ Headers: Content-Type: application/json
│       └─ Body: {"headline": "...", "article_text": "..."}
│
├─▶ Backend (FastAPI)
│   │
│   ├─ Validation (Pydantic)
│   │   └─ Verify schema: headline & article_text required
│   │
│   ├─ REQUEST PROCESSING
│   │   │
│   │   ├─ [STEP 1] Parallel Processing ───────────────────────┐
│   │   │                                                        │
│   │   │   BRANCH A: ML Classification                         │
│   │   │   └─ ml_service.predict()                             │
│   │   │      ├─ Load pipeline (TF-IDF)                        │
│   │   │      ├─ Combine headline + article_text               │
│   │   │      ├─ Vectorize to TF-IDF features                  │
│   │   │      ├─ Pass to Logistic Regression                   │
│   │   │      └─ Return: prediction (REAL/FAKE), confidence    │
│   │   │                                                        │
│   │   │   BRANCH B: Claim Extraction                          │
│   │   │   └─ claim_extractor.extract()                        │
│   │   │      ├─ Segment text into sentences                   │
│   │   │      ├─ Filter boilerplate & opinions                 │
│   │   │      ├─ Detect numeric facts                          │
│   │   │      ├─ Score by factuality                           │
│   │   │      └─ Return: top N claims (default: 5)             │
│   │   │                                                        │
│   │   │   BRANCH C: Evidence Retrieval (Concurrent)           │
│   │   │   └─ evidence_retriever.retrieve()                    │
│   │   │      ├─ For each claim:                               │
│   │   │      │  ├─▶ google_factcheck_provider.search()        │
│   │   │      │  ├─▶ web_search_provider.search()              │
│   │   │      │  │   ├─ Tavily API                             │
│   │   │      │  │   ├─ SerpAPI                                │
│   │   │      │  │   └─ Bing API                               │
│   │   │      │  └─▶ wikipedia_provider.search()               │
│   │   │      │                                                 │
│   │   │      ├─ Deduplicate results                            │
│   │   │      ├─ Rank by relevance                              │
│   │   │      └─ Return: evidence items with sources            │
│   │   │                                                        │
│   │   └─ [AWAIT ALL BRANCHES] ────────────────────────────────┘
│   │
│   ├─ [STEP 2] Claim Verification
│   │   └─ claim_verification_service.verify()
│   │      ├─ For each claim:
│   │      │  ├─ Compare against evidence
│   │      │  ├─ Calculate relevance (lexical overlap)
│   │      │  ├─ Assign stance: SUPPORTED/CONTRADICTED/INSUFFICIENT
│   │      │  └─ Score confidence
│   │      │
│   │      └─ Return: verification results per claim
│   │
│   ├─ [STEP 3] Decision Engine
│   │   └─ decision_engine.decide()
│   │      ├─ Calculate ML signal: prediction × confidence
│   │      ├─ Calculate Evidence signal:
│   │      │  └─ (SUPPORTED count - CONTRADICTED count) / total claims
│   │      │
│   │      ├─ Hybrid Decision:
│   │      │  └─ final_decision = (ML_score × 0.35) + (Evidence_score × 0.65)
│   │      │
│   │      └─ Return: final_decision (REAL/FAKE), confidence
│   │
│   ├─ [STEP 4] Explanation Generation
│   │   └─ explanation_service.generate()
│   │      ├─ Summarize ML prediction
│   │      ├─ Summarize supported/contradicted claims
│   │      ├─ Cite evidence sources
│   │      └─ Return: human-readable explanation (Markdown)
│   │
│   ├─ [STEP 5] Database Persistence (if MongoDB available)
│   │   └─ repositories.save_analysis()
│   │      ├─ analysis_repository.save(analysis_data)
│   │      ├─ claim_repository.save_many(claims)
│   │      └─ evidence_repository.save_many(evidence)
│   │
│   ├─ RESPONSE BUILDING
│   │   └─ CompleteAnalysisResponse
│   │      ├─ analysis_id: UUID
│   │      ├─ ml_decision: str
│   │      ├─ ml_confidence: float
│   │      ├─ claims: List[str]
│   │      ├─ evidence: List[EvidenceItem]
│   │      ├─ verifications: List[VerificationResult]
│   │      ├─ decision_factors: dict
│   │      ├─ explanation: str
│   │      ├─ final_decision: str
│   │      └─ created_at: datetime
│   │
│   └─ HTTP 200 Response
│       └─ Content-Type: application/json
│           Body: {...complete analysis JSON...}
│
└─▶ Frontend (React)
    │
    ├─ Parse JSON response
    ├─ Update state
    │
    └─ RENDER RESULTS PAGE
        ├─ Decision Card
        │   ├─ Icon (CheckCircle2 or ShieldAlert)
        │   ├─ Label (REAL or FAKE)
        │   └─ Confidence: 92.3%
        │
        ├─ Claims Section
        │   └─ For each claim:
        │       ├─ Claim text
        │       ├─ Verification status badge
        │       └─ Supporting evidence list
        │
        ├─ Evidence Section
        │   └─ For each evidence item:
        │       ├─ Source name + URL
        │       ├─ Relevance score
        │       └─ Snippet text
        │
        └─ Explanation Section
            └─ Human-readable rationale (Markdown)
```

---

## TECHNOLOGY STACK

### Layer-by-Layer Technology Breakdown

```
TIER 1: PRESENTATION (Browser)
┌─────────────────────────────────────────┐
│ Runtime Environment: Node.js 16+ (Dev)  │
│ Browser: Chrome/Firefox/Edge (Runtime)  │
├─────────────────────────────────────────┤
│ Framework:        React 19.2.8          │
│ Module Bundler:   Vite 8.2.0            │
│ Router:           React Router 7.18.2   │
│ HTTP Client:      Axios 1.19.0          │
│ Component Library: Lucide React 1.31.0  │
│ CSS:              Custom (index.css)    │
├─────────────────────────────────────────┤
│ Build Output: dist/ (static SPA)        │
│ Dev Server: http://localhost:5173      │
└─────────────────────────────────────────┘

TIER 2: APPLICATION SERVER (Backend)
┌─────────────────────────────────────────┐
│ Runtime Environment: Python 3.10+       │
│ Virtual Environment: venv or conda      │
├─────────────────────────────────────────┤
│ Web Framework:    FastAPI 0.110+        │
│ ASGI Server:      Uvicorn 0.28+         │
│ Concurrency:      asyncio (built-in)    │
├─────────────────────────────────────────┤
│ Data Validation:  Pydantic 2.6+         │
│ Config Management: pydantic-settings    │
│ HTTP Client:      httpx 0.27+ (async)   │
│ Web Scraping:     trafilatura (latest)  │
│ Logging:          logging (stdlib)      │
├─────────────────────────────────────────┤
│ Server: http://127.0.0.1:8000          │
│ API Prefix: /api                        │
└─────────────────────────────────────────┘

TIER 3: ML & ANALYTICS
┌─────────────────────────────────────────┐
│ ML Framework:     scikit-learn 1.4+     │
│ Vectorizer:       TF-IDF (sklearn)      │
│ Classifier:       LogisticRegression    │
├─────────────────────────────────────────┤
│ Data Processing:  pandas 2.0+           │
│ Numerical Ops:    numpy 1.25+           │
│ Visualization:    matplotlib 3.7+       │
│ Serialization:    joblib 1.3+           │
├─────────────────────────────────────────┤
│ Model Artifact:   pipeline.joblib       │
│ Vectorizer File:  vectorizer.joblib     │
│ Metrics File:     metrics.json          │
└─────────────────────────────────────────┘

TIER 4: DATABASE (Optional)
┌─────────────────────────────────────────┐
│ Database:         MongoDB 4.0+          │
│ Async Driver:     Motor 3.3+            │
│ Sync Driver:      PyMongo 4.6+          │
├─────────────────────────────────────────┤
│ Collections:                            │
│ • analyses         - Analysis results   │
│ • claims           - Extracted claims   │
│ • evidence         - Evidence items     │
│ • feedback         - User feedback      │
├─────────────────────────────────────────┤
│ Indexes:                                │
│ • analyses(user_id)                     │
│ • analyses(created_at)                  │
│ • analyses(final_decision)              │
├─────────────────────────────────────────┤
│ Deploy: Local (27017) or Atlas Cloud    │
└─────────────────────────────────────────┘

TIER 5: EXTERNAL SERVICES
┌─────────────────────────────────────────┐
│ Evidence Providers (via httpx):         │
│ • Google Fact Check API                 │
│ • Tavily Search                         │
│ • SerpAPI                               │
│ • Bing Search API                       │
│ • Wikipedia API                         │
└─────────────────────────────────────────┘
```

---

## BACKEND ARCHITECTURE

### File Organization & Responsibilities

```
backend/app/
│
├─ main.py ⭐ [ENTRY POINT]
│  ├─ Creates FastAPI application
│  ├─ Configures CORS middleware
│  ├─ Manages lifespan (startup/shutdown)
│  ├─ Loads ML pipeline on startup
│  ├─ Connects to MongoDB on startup
│  └─ Registers all route routers
│
├─ config.py [CONFIGURATION]
│  ├─ Settings class (Pydantic)
│  ├─ Environment variable parsing
│  ├─ Model path resolution
│  ├─ CORS origin configuration
│  ├─ MongoDB connection settings
│  └─ API key management
│
├─ api/ [ROUTE HANDLERS]
│  ├─ routes_analysis.py
│  │  ├─ POST /api/analyze          - Text analysis
│  │  ├─ POST /api/analyze/url      - URL analysis
│  │  └─ POST /api/analyze/claim    - Claim-only analysis
│  │
│  ├─ routes_metrics.py
│  │  └─ GET /api/metrics           - Model metrics
│  │
│  ├─ routes_health.py
│  │  └─ GET /api/health            - Health check
│  │
│  ├─ routes_history.py
│  │  ├─ GET /api/history           - Get all analyses
│  │  ├─ GET /api/history/{id}      - Get single analysis
│  │  └─ DELETE /api/history/{id}   - Delete analysis
│  │
│  └─ routes_feedback.py
│     └─ POST /api/feedback         - Store user feedback
│
├─ services/ [BUSINESS LOGIC - Core Processing]
│  ├─ ml_service.py ⭐ [ML INFERENCE]
│  │  ├─ MLService class
│  │  ├─ load_model() - Load pipeline from joblib
│  │  └─ predict(headline, article_text) - Run inference
│  │
│  ├─ claim_extractor.py [CLAIM IDENTIFICATION]
│  │  ├─ ClaimExtractor class
│  │  ├─ extract(article_text) - Find verifiable claims
│  │  ├─ _segment_sentences()
│  │  ├─ _is_boilerplate()
│  │  └─ _score_factuality()
│  │
│  ├─ evidence_retriever.py [EVIDENCE COLLECTION]
│  │  ├─ EvidenceRetriever class
│  │  ├─ retrieve(claims) - Get evidence from all sources
│  │  ├─ _query_google_factcheck()
│  │  ├─ _query_web_search()
│  │  ├─ _query_wikipedia()
│  │  └─ _deduplicate_and_rank()
│  │
│  ├─ claim_verification_service.py [CLAIM VERIFICATION]
│  │  ├─ ClaimVerificationService class
│  │  ├─ verify(claims, evidence)
│  │  ├─ _calculate_relevance_score()
│  │  ├─ _assign_stance()
│  │  └─ _aggregate_confidence()
│  │
│  ├─ decision_engine.py [HYBRID DECISION]
│  │  ├─ DecisionEngine class
│  │  ├─ decide(ml_prediction, evidence_verification)
│  │  ├─ _calculate_ml_signal()
│  │  ├─ _calculate_evidence_signal()
│  │  └─ Formula: final = (ML × 0.35) + (Evidence × 0.65)
│  │
│  ├─ explanation_service.py [RESULT EXPLANATION]
│  │  ├─ ExplanationService class
│  │  ├─ generate(analysis_data)
│  │  ├─ _summarize_ml_prediction()
│  │  ├─ _summarize_claims()
│  │  └─ _cite_evidence()
│  │
│  ├─ article_url_service.py [URL EXTRACTION]
│  │  ├─ ArticleURLService class
│  │  ├─ extract(url) - Get headline & body from URL
│  │  ├─ _fetch_html()
│  │  ├─ _parse_with_trafilatura()
│  │  └─ _extract_metadata()
│  │
│  ├─ source_scoring_service.py [SOURCE CREDIBILITY]
│  │  ├─ SourceScoringService class
│  │  └─ score_source(source_name, source_url)
│  │
│  ├─ evidence_analyzer.py [EVIDENCE ANALYSIS]
│  │  ├─ EvidenceAnalyzer class
│  │  ├─ analyze(evidence_list)
│  │  ├─ _categorize_by_stance()
│  │  └─ _calculate_aggregate_score()
│  │
│  └─ ml_loader.py [ML MODEL LOADING]
│     └─ Helper functions for model initialization
│
├─ database/ [DATA PERSISTENCE]
│  ├─ connection.py
│  │  ├─ MongoDB class
│  │  ├─ connect(uri, database_name) - Async connect
│  │  ├─ disconnect() - Cleanup
│  │  ├─ create_indexes() - Index creation
│  │  └─ database property
│  │
│  └─ repositories.py
│     ├─ AnalysisRepository class
│     │  ├─ save(analysis_data)
│     │  ├─ get_by_id(analysis_id)
│     │  ├─ get_by_user(user_id)
│     │  └─ delete(analysis_id)
│     │
│     ├─ ClaimRepository class
│     │  ├─ save_many(claims)
│     │  ├─ get_by_analysis(analysis_id)
│     │  └─ delete_by_analysis(analysis_id)
│     │
│     └─ EvidenceRepository class
│        ├─ save_many(evidence_items)
│        ├─ get_by_claim(claim_id)
│        └─ delete_by_analysis(analysis_id)
│
├─ models/ [DATA SCHEMAS]
│  └─ schemas.py
│     ├─ Input Schemas:
│     │  ├─ AnalysisInput
│     │  ├─ URLAnalysisRequest
│     │  └─ ClaimAnalysisRequest
│     │
│     ├─ Output Schemas:
│     │  ├─ MLAnalyzeResponse
│     │  ├─ VerificationResult
│     │  ├─ EvidenceItem
│     │  ├─ CompleteAnalysisResponse
│     │  └─ HealthResponse
│     │
│     └─ Helper Schemas:
│        ├─ ErrorResponse
│        └─ ClaimAnalysisResponse
│
├─ providers/ [EXTERNAL API WRAPPERS]
│  ├─ base_provider.py
│  │  └─ BaseProvider (abstract base class)
│  │     └─ search(query) - Abstract method
│  │
│  ├─ google_factcheck.py
│  │  ├─ GoogleFactCheckProvider class
│  │  └─ search(query) - Google Fact Check API
│  │
│  ├─ web_search.py
│  │  ├─ TavilyProvider class
│  │  ├─ SerpAPIProvider class
│  │  ├─ BingSearchProvider class
│  │  └─ All implement search(query)
│  │
│  └─ wikipedia_provider.py
│     ├─ WikipediaProvider class
│     └─ search(query) - Wikipedia lookup (free)
│
└─ utils/ [UTILITIES]
   └─ logger.py
      ├─ Logger configuration
      ├─ get_logger() - Get named logger
      └─ Structured logging setup
```

---

## FRONTEND ARCHITECTURE

### Component & File Organization

```
frontend/
│
├─ package.json [DEPENDENCIES & SCRIPTS]
│  ├─ Scripts:
│  │  ├─ npm run dev      - Start dev server (Vite)
│  │  ├─ npm run build    - Build for production
│  │  ├─ npm run preview  - Preview production build
│  │  └─ npm run lint     - Check code quality
│  │
│  └─ Dependencies:
│     ├─ react 19.2.8
│     ├─ react-dom 19.2.8
│     ├─ react-router-dom 7.18.2
│     ├─ axios 1.19.0
│     ├─ lucide-react 1.31.0
│     └─ vite 8.2.0 (dev)
│
├─ vite.config.js [BUNDLER CONFIGURATION]
│  ├─ React plugin enabled
│  ├─ Dev server config
│  └─ Build optimizations
│
├─ index.html [HTML ENTRY POINT]
│  └─ Links to main.jsx
│
├─ src/
│  │
│  ├─ main.jsx [REACT INITIALIZATION]
│  │  └─ Creates React root & mounts App component
│  │
│  ├─ App.jsx ⭐ [MAIN APP COMPONENT]
│  │  ├─ Navbar Component
│  │  │  ├─ Brand (logo + title)
│  │  │  └─ Navigation Links
│  │  │
│  │  ├─ BrowserRouter Setup
│  │  │  ├─ Route: /         → Dashboard
│  │  │  └─ Route: /analyze  → Analyze Page
│  │  │
│  │  ├─ Dashboard Component
│  │  │  ├─ Fetches metrics from API
│  │  │  ├─ useEffect hook for async data
│  │  │  ├─ useState for loading/error/data
│  │  │  └─ Displays model performance
│  │  │
│  │  └─ Error Boundary (implicit)
│  │
│  ├─ pages/
│  │  └─ Analyze.jsx [MAIN ANALYSIS PAGE]
│  │     ├─ State Management:
│  │     │  ├─ analysisMode (TEXT or URL)
│  │     │  ├─ headline / articleText / url
│  │     │  ├─ analysis result
│  │     │  ├─ loading / error states
│  │     │  └─ selectedClaimIndex
│  │     │
│  │     ├─ Form Section:
│  │     │  ├─ Input mode toggle (Text vs URL)
│  │     │  ├─ Text inputs with validation
│  │     │  ├─ Analyze button
│  │     │  └─ Error messages display
│  │     │
│  │     ├─ Results Section (conditional rendering):
│  │     │  ├─ Decision Card
│  │     │  │  ├─ REAL/FAKE badge
│  │     │  │  ├─ Confidence score
│  │     │  │  └─ Explanation text
│  │     │  │
│  │     │  ├─ Claims List
│  │     │  │  └─ For each claim:
│  │     │  │     ├─ Claim text
│  │     │  │     ├─ Verification status
│  │     │  │     └─ Evidence section
│  │     │  │
│  │     │  └─ Evidence Details
│  │     │     └─ Source links & citations
│  │     │
│  │     ├─ Event Handlers:
│  │     │  ├─ handleAnalyzeArticle()
│  │     │  ├─ handleAnalyzeUrl()
│  │     │  ├─ handleSelectClaim()
│  │     │  └─ API call coordination
│  │     │
│  │     └─ UI Components:
│  │        ├─ Input fields
│  │        ├─ Loading spinner
│  │        ├─ Error alerts
│  │        └─ Responsive layout
│  │
│  ├─ services/
│  │  └─ api.js [API CLIENT - Axios Wrapper]
│  │     ├─ Base URL: http://127.0.0.1:8000/api
│  │     │
│  │     ├─ Analysis Functions:
│  │     │  ├─ analyzeArticle(headline, articleText)
│  │     │  │  └─ POST /analyze
│  │     │  │
│  │     │  └─ analyzeArticleUrl(url)
│  │     │     └─ POST /analyze/url
│  │     │
│  │     ├─ Metrics Functions:
│  │     │  └─ getModelMetrics()
│  │     │     └─ GET /metrics
│  │     │
│  │     ├─ History Functions:
│  │     │  ├─ getAnalysisHistory(userId, limit)
│  │     │  │  └─ GET /history
│  │     │  │
│  │     │  └─ getAnalysisById(analysisId)
│  │     │     └─ GET /history/{id}
│  │     │
│  │     └─ Error Handling:
│  │        └─ Try-catch with fallback messages
│  │
│  ├─ index.css [GLOBAL STYLES]
│  │  ├─ CSS Variables (dark theme)
│  │  ├─ Navbar styling
│  │  ├─ Page layouts
│  │  ├─ Form inputs
│  │  ├─ Cards & sections
│  │  ├─ Decision badges
│  │  ├─ Evidence lists
│  │  └─ Responsive design (mobile-first)
│  │
│  └─ assets/
│     └─ Images, icons (if any)
│
└─ public/
   ├─ index.html [Fallback HTML]
   ├─ favicon.svg
   └─ icons.svg
```

---

## SERVICE DEPENDENCIES

### Service Call Graph

```
API ENDPOINT: POST /api/analyze
│
└─▶ routes_analysis.py
    ├─▶ article_url_service.extract() [if URL]
    │   └─▶ trafilatura library
    │
    ├─▶ [PARALLEL] Claim Extraction
    │   └─▶ claim_extractor.extract()
    │       ├─ text_clean utilities
    │       └─ Returns: list[str]
    │
    ├─▶ [PARALLEL] ML Service
    │   └─▶ ml_service.predict()
    │       ├─ Load pipeline.joblib
    │       ├─ TF-IDF vectorizer
    │       ├─ Logistic Regression classifier
    │       └─ Returns: {prediction, confidence, probability}
    │
    ├─▶ [PARALLEL] Evidence Retrieval
    │   └─▶ evidence_retriever.retrieve()
    │       ├─▶ google_factcheck_provider.search()
    │       │   └─▶ httpx (async) → Google API
    │       │
    │       ├─▶ web_search_provider.search()
    │       │   ├─▶ TavilyProvider → Tavily API
    │       │   ├─▶ SerpAPIProvider → SerpAPI
    │       │   └─▶ BingSearchProvider → Bing API
    │       │
    │       ├─▶ wikipedia_provider.search()
    │       │   └─▶ httpx (async) → Wikipedia API
    │       │
    │       └─ Evidence Deduplication & Ranking
    │
    ├─▶ claim_verification_service.verify()
    │   ├─ source_scoring_service.score_source()
    │   └─ Returns: list[VerificationResult]
    │
    ├─▶ decision_engine.decide()
    │   └─ Hybrid: (ML × 0.35) + (Evidence × 0.65)
    │
    ├─▶ explanation_service.generate()
    │   └─ Returns: human-readable explanation
    │
    ├─▶ [OPTIONAL] MongoDB Persistence
    │   ├─▶ analysis_repository.save()
    │   │   └─ Motor → analyses collection
    │   │
    │   ├─▶ claim_repository.save_many()
    │   │   └─ Motor → claims collection
    │   │
    │   └─▶ evidence_repository.save_many()
    │       └─ Motor → evidence collection
    │
    └─▶ Return CompleteAnalysisResponse (JSON)

API ENDPOINT: GET /api/metrics
│
└─▶ routes_metrics.py
    ├─ Read outputs/metrics.json
    └─ Parse & return metrics

API ENDPOINT: GET /api/health
│
└─▶ routes_health.py
    ├─ Check ml_service.is_loaded
    ├─ Check mongodb.connected
    └─ Return health status

API ENDPOINT: GET /api/history
│
└─▶ routes_history.py
    └─▶ analysis_repository.get_by_user()
        └─ Motor → MongoDB query
```

---

## DATABASE SCHEMA

### MongoDB Collections Structure

```
DATABASE: fake_news_detection
│
├─ Collection: analyses
│  │
│  ├─ Indexes:
│  │  ├─ _id (ObjectId, primary)
│  │  ├─ user_id (for history queries)
│  │  ├─ created_at (for sorting)
│  │  └─ final_decision (for filtering)
│  │
│  └─ Document Structure:
│     {
│       "_id": ObjectId,
│       "analysis_id": "uuid-string",
│       "user_id": "anonymous",
│       "created_at": ISODate,
│       "input": {
│         "headline": "...",
│         "article_text": "..."
│       },
│       "ml_result": {
│         "prediction": "REAL",
│         "confidence": 0.924,
│         "probability": [0.076, 0.924]
│       },
│       "claims": [...array of claim IDs...],
│       "evidence": [...array of evidence IDs...],
│       "verifications": [
│         {
│           "claim": "...",
│           "stance": "SUPPORTED",
│           "confidence": 0.85
│         }
│       ],
│       "decision_factors": {
│         "ml_signal": 0.924,
│         "evidence_signal": 0.78,
│         "final": 0.88
│       },
│       "final_decision": "REAL",
│       "confidence": 0.88,
│       "explanation": "...",
│       "processing_time_ms": 4521
│     }
│
├─ Collection: claims
│  │
│  └─ Document Structure:
│     {
│       "_id": ObjectId,
│       "claim_id": "uuid-string",
│       "analysis_id": "uuid-string",
│       "text": "The claim statement",
│       "rank": 1,
│       "factuality_score": 0.85,
│       "created_at": ISODate
│     }
│
├─ Collection: evidence
│  │
│  └─ Document Structure:
│     {
│       "_id": ObjectId,
│       "evidence_id": "uuid-string",
│       "claim_id": "uuid-string",
│       "analysis_id": "uuid-string",
│       "source_name": "Google Fact Check",
│       "source_url": "https://...",
│       "title": "Fact check article title",
│       "snippet": "Evidence text excerpt",
│       "relevance_score": 0.92,
│       "stance": "SUPPORTED",
│       "created_at": ISODate
│     }
│
└─ Collection: feedback
   │
   └─ Document Structure:
      {
        "_id": ObjectId,
        "feedback_id": "uuid-string",
        "analysis_id": "uuid-string",
        "user_feedback": "This analysis was helpful/incorrect",
        "correct_label": "REAL",
        "created_at": ISODate
      }
```

---

## API ENDPOINTS MAP

### RESTful API Structure

```
BASE_URL: http://127.0.0.1:8000/api

┌─────────────────────────────────────────────────────────────┐
│ ANALYSIS ENDPOINTS                                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ POST /analyze                                              │
│ ├─ Purpose: Analyze article text                           │
│ ├─ Request Body:                                           │
│ │  {                                                       │
│ │    "headline": "Article headline",                       │
│ │    "article_text": "Full article text"                   │
│ │  }                                                       │
│ ├─ Response: CompleteAnalysisResponse (JSON)               │
│ └─ Status: 200 (OK) | 400 (Invalid) | 422 (Validation)    │
│                                                             │
│ POST /analyze/url                                          │
│ ├─ Purpose: Analyze article from URL                       │
│ ├─ Request Body:                                           │
│ │  {                                                       │
│ │    "url": "https://news.example.com/article"             │
│ │  }                                                       │
│ ├─ Response: CompleteAnalysisResponse (JSON)               │
│ └─ Status: 200 (OK) | 400 (Invalid) | 422 (Validation)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ METRICS ENDPOINTS                                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ GET /metrics                                               │
│ ├─ Purpose: Get model performance metrics                  │
│ ├─ Query Parameters: (none)                                │
│ ├─ Response:                                               │
│ │  {                                                       │
│ │    "model": "TF-IDF + Logistic Regression",             │
│ │    "accuracy": 0.9964,                                   │
│ │    "precision": 0.9943,                                  │
│ │    "recall": 0.9985,                                     │
│ │    "f1_score": 0.9964,                                   │
│ │    "test_samples": 2556                                  │
│ │  }                                                       │
│ └─ Status: 200 (OK) | 404 (Metrics file missing)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ HEALTH & STATUS ENDPOINTS                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ GET /health                                                │
│ ├─ Purpose: Check service health & readiness               │
│ ├─ Query Parameters: (none)                                │
│ ├─ Response:                                               │
│ │  {                                                       │
│ │    "status": "healthy",                                  │
│ │    "service": "Fake News Detection API",                │
│ │    "version": "1.0.0",                                   │
│ │    "ml_model_loaded": true,                              │
│ │    "model_path": "/path/to/pipeline.joblib",            │
│ │    "database": "connected" | "unavailable"               │
│ │  }                                                       │
│ └─ Status: 200 (OK)                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ HISTORY ENDPOINTS (Requires MongoDB)                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ GET /history                                               │
│ ├─ Purpose: Get user's analysis history                    │
│ ├─ Query Parameters:                                       │
│ │  ├─ user_id: string (default: "anonymous")              │
│ │  └─ limit: int (default: 20)                             │
│ ├─ Response: List[Analysis]                                │
│ └─ Status: 200 (OK) | 503 (DB unavailable)                │
│                                                             │
│ GET /history/{id}                                          │
│ ├─ Purpose: Get specific analysis by ID                    │
│ ├─ Path Parameters:                                        │
│ │  └─ id: analysis_id (UUID)                               │
│ ├─ Response: Analysis object                               │
│ └─ Status: 200 (OK) | 404 (Not found) | 503 (DB down)     │
│                                                             │
│ DELETE /history/{id}                                       │
│ ├─ Purpose: Delete specific analysis                       │
│ ├─ Path Parameters:                                        │
│ │  └─ id: analysis_id (UUID)                               │
│ ├─ Response: {"status": "deleted"}                         │
│ └─ Status: 200 (OK) | 404 (Not found) | 503 (DB down)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ FEEDBACK ENDPOINTS                                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ POST /feedback                                             │
│ ├─ Purpose: Submit user feedback on analysis               │
│ ├─ Request Body:                                           │
│ │  {                                                       │
│ │    "analysis_id": "uuid",                                │
│ │    "user_feedback": "Feedback text",                     │
│ │    "correct_label": "REAL" | "FAKE"                      │
│ │  }                                                       │
│ ├─ Response: {"status": "feedback_received"}               │
│ └─ Status: 200 (OK) | 400 (Invalid) | 422 (Validation)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## DEPLOYMENT ARCHITECTURE

### Local Development Setup

```
DEVELOPER MACHINE
│
├─ Terminal 1: Backend Server
│  ├─ Command: cd backend && python -m uvicorn app.main:app --reload
│  ├─ Port: 8000
│  ├─ Hot Reload: Yes
│  ├─ Logs: Console output
│  └─ Database: mongodb://localhost:27017 (optional)
│
├─ Terminal 2: Frontend Dev Server
│  ├─ Command: cd frontend && npm run dev
│  ├─ Port: 5173
│  ├─ Hot Module Reload: Yes
│  ├─ Logs: Console + Browser DevTools
│  └─ Backend URL: http://127.0.0.1:8000
│
└─ Browser
   └─ http://localhost:5173
      ├─ Dashboard page
      └─ Analyze page
```

### Production Deployment (Conceptual)

```
┌─────────────────────────────────────────────┐
│         PRODUCTION ENVIRONMENT              │
├─────────────────────────────────────────────┤
│                                             │
│  ┌─ Frontend (Static SPA) ────────────┐    │
│  │ • CDN (CloudFlare/AWS CloudFront)  │    │
│  │ • HTTPS enabled                    │    │
│  │ • SPA served from dist/            │    │
│  │ • API calls to /api/*              │    │
│  └─────────────────────────────────────┘    │
│              ↓                               │
│  ┌─ Backend (FastAPI) ────────────────┐    │
│  │ • Docker container                 │    │
│  │ • Deployed on cloud (AWS/Azure)    │    │
│  │ • Gunicorn + Uvicorn (workers)     │    │
│  │ • Environment vars from secrets    │    │
│  │ • HTTPS/TLS enabled                │    │
│  └─────────────────────────────────────┘    │
│              ↓                               │
│  ┌─ Database (MongoDB) ───────────────┐    │
│  │ • MongoDB Atlas (cloud)            │    │
│  │ • or self-hosted MongoDB           │    │
│  │ • Backups enabled                  │    │
│  │ • Connection pooling               │    │
│  └─────────────────────────────────────┘    │
│              ↓                               │
│  ┌─ External APIs ────────────────────┐    │
│  │ • API keys from secrets manager    │    │
│  │ • Rate limiting handled            │    │
│  │ • Fallback to free tier (Wikipedia)│    │
│  └─────────────────────────────────────┘    │
│                                             │
└─────────────────────────────────────────────┘
```

---

## SUMMARY: Key Architecture Points

### ✅ Layered Design
- **Presentation:** React SPA (stateless, component-based)
- **Application:** FastAPI (async, route-based, service-oriented)
- **Business Logic:** Service layer (claim extraction, verification, ML)
- **Data Access:** Repository pattern (MongoDB, filesystem)
- **External:** Provider pattern (Google, Web, Wikipedia)

### ✅ Separation of Concerns
- Routes only handle HTTP
- Services handle business logic
- Repositories handle data access
- Providers handle external APIs
- Models define data contracts

### ✅ Async First
- FastAPI with asyncio
- Motor for async MongoDB
- httpx for async HTTP requests
- Concurrent evidence retrieval

### ✅ Extensible Design
- Abstract BaseProvider for new evidence sources
- Service interfaces allow easy testing
- Configuration-driven settings

### ✅ Production Ready
- Error handling & validation
- Logging throughout
- Optional MongoDB (graceful degradation)
- CORS configured
- Health checks
- Metrics tracking

