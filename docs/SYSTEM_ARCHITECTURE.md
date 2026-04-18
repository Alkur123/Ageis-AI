# Aegis AI — Complete System Architecture
**As-built | April 2026 | Chakravyuha V2.5 Demo**

---

## Full System Flow

```mermaid
flowchart TD
    %% Styling
    classDef aws fill:#f97316,stroke:#ea580c,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef storage fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef compute fill:#10b981,stroke:#059669,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef database fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef proxy fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#fff,rx:8px,ry:8px;

    subgraph AWS[☁️ AWS INFRASTRUCTURE]
        CDN[🌐 AWS CloudFront CDN<br/>Global edge · HTTPS · DDoS protection]:::aws
        
        S3[📦 AWS S3 Bucket<br/>React + Vite Build]:::storage
        APIProxy[🔄 CloudFront API Proxy<br/>Forwards to EC2]:::proxy
        
        EC2[🚀 AWS EC2 Instance<br/>Docker Container · Gunicorn + Uvicorn · FastAPI]:::compute
        
        Mongo[(🗄️ MongoDB<br/>Audit Logs)]:::database
        Redis[(⚡ Redis<br/>Sessions)]:::database
        Groq[🧠 Groq API / Ollama<br/>LLM Processing]:::compute
        
        CDN -- "Static assets" --> S3
        CDN -- "/api/* requests" --> APIProxy
        APIProxy --> EC2
        
        EC2 --> Mongo
        EC2 --> Redis
        EC2 --> Groq
    end
```

---

## Frontend Architecture (React + Vite)

```mermaid
flowchart LR
    %% Styling
    classDef browser fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef view fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef api fill:#10b981,stroke:#059669,stroke-width:2px,color:#fff,rx:8px,ry:8px;

    Browser[🌐 Browser]:::browser --> App[📱 App.jsx<br/>Main Layout]:::view
    App --> LeftPanel[💬 LEFT PANEL<br/>Chat Interface]:::view
    App --> RightPanel[📊 RIGHT PANEL<br/>Intelligence]:::view
    
    LeftPanel --> Input[⌨️ User Input]
    LeftPanel --> Thread[🗨️ Message Thread<br/>AI Response Cards]
    LeftPanel --> Banner[🚨 Decision Banner]
    
    RightPanel --> RiskTab[⚠️ RISK TAB<br/>Gauge, Session, Graph]
    RightPanel --> EvidTab[🔍 EVIDENCE TAB<br/>Reason, Signal, FAISS]
    RightPanel --> MonTab[📈 MONITOR TAB<br/>Metrics, Heatmap, Audits]
    
    App --> API[🔌 api.js<br/>HTTP Client]:::api
    API --> Endpoints[POST /analyze<br/>POST /reset-session<br/>GET /metrics...]:::api
```

---

## Backend Architecture (FastAPI)

```mermaid
flowchart TD
    %% Styling
    classDef core fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef middle fill:#f59e0b,stroke:#d97706,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef init fill:#14b8a6,stroke:#0d9488,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef route fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef db fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#fff,rx:8px,ry:8px;

    FastAPI[⚙️ FastAPI Server<br/>server.py]:::core --> Middleware[🛡️ Middleware Stack<br/>CORS · Rate Limiter]:::middle
    FastAPI --> Startup[🚀 Startup Event<br/>Load Engines]:::init
    FastAPI --> Routes[🛣️ API Routes]:::route
    FastAPI --> Mongo[(🗄️ MongoDB Connection)]:::db
    
    Routes --> R1[POST /analyze]
    Routes --> R2[GET /health & /stats]
    Routes --> R3[GET /session & /audit]
```

---

## Governance Pipeline — 8 Stages (pipeline.py)

```mermaid
flowchart TD
    %% Styling
    classDef input fill:#3b82f6,stroke:#2563eb,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef stage fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef final fill:#10b981,stroke:#059669,stroke-width:2px,color:#fff,rx:8px,ry:8px;
    classDef outcome fill:#ec4899,stroke:#db2777,stroke-width:2px,color:#fff,rx:8px,ry:8px;

    User[👤 User Query raw]:::input --> S1[🛡️ STAGE 1: PII Redaction<br/>Regex Scan ➔ REDACTED Query]:::stage
    S1 --> S2[🧠 STAGE 2: FAISS Semantic Search<br/>ONNX Embed ➔ IndexFlatIP]:::stage
    S2 --> S3[⚔️ STAGE 3: Attack Vector Analysis<br/>Keyword Detection]:::stage
    S3 --> S4[🔍 STAGE 4: Category Resolution<br/>Semantic + Overrides]:::stage
    S4 --> S5[⏳ STAGE 5: Session Memory<br/>Redis ➔ Cumulative Risk]:::stage
    S5 --> S6[⚖️ STAGE 6: Policy Engine<br/>Multi-signal Scoring]:::stage
    S6 --> S7[📝 STAGE 7: Decision Trace<br/>Causal Explainability]:::stage
    
    S7 --> Decision{Decision Outcome}
    
    Decision -- BLOCK / SUPPORT --> Hardcoded[🛑 Hardcoded Safe Responses<br/>Crisis / Regulatory]:::outcome
    Decision -- ALLOW --> S8[💬 STAGE 8: LLM Response<br/>Groq ➔ Ollama Fallback]:::final
    
    Hardcoded --> Audit[🗄️ MongoDB Audit Write]
    S8 --> Audit
    
    Audit --> Frontend[🖥️ Full JSON Response to Frontend]:::input
```

---

## Data Flow Summary

```mermaid
flowchart LR
    %% Styling
    classDef step fill:#8b5cf6,stroke:#7c3aed,stroke-width:2px,color:#fff,rx:8px,ry:8px;

    In((User Input)) --> Flow
    
    subgraph Flow[Pipeline < 1ms]
        direction TB
        1[1. PII Stripped]:::step --> 2[2. ONNX Embed]:::step
        2 --> 3[3. FAISS Search]:::step
        3 --> 4[4. Attack Detect]:::step
        4 --> 5[5. Category]:::step
        5 --> 6[6. Session]:::step
        6 --> 7[7. Policy]:::step
        7 --> 8[8. Trace]:::step
    end
    
    Flow --> Outcome{Action}
    Outcome -- BLOCK --> Resp[Safe Response<br/>0ms LLM]
    Outcome -- ALLOW --> LLM[Groq API<br/>100-200ms]
    
    Resp --> Audit[MongoDB Audit]
    LLM --> Audit
```

---

## Data Stores

| Store | Technology | What It Holds | Durability |
|-------|-----------|---------------|------------|
| Audit Logs | MongoDB (motor async) | Every decision with full trace | Persistent |
| Session State | Redis → in-memory fallback | Per-user risk scores, history, distress | 24hr TTL |
| Vector Index | FAISS (in-memory) | 998 pre-computed embeddings | Rebuilt on startup |
| ONNX Model | File (onnx_model/) | Query embedding weights | Static file |
| LLM Response Cache | In-memory dict | Query → structured response | 5 min TTL |
| Policy Dataset | MongoDB + CSV/JSONL | Training data (downloadable) | Persistent |

---

## API Endpoints (Complete)

| Method | Path | Purpose | Auth |
|--------|------|---------|------|
| POST | /api/analyze | Main governance — analyze a query | Rate-limited (60/min) |
| GET | /api/health | Liveness + engine ready flag | None |
| GET | /api/stats | FAISS stats (vectors, categories) | None |
| GET | /api/metrics | Request counters + avg latency | None |
| GET | /api/session/{id} | Raw session state | None |
| POST | /api/reset-session | Clear session | None |
| GET | /api/audit | Last 50 audit log entries (MongoDB) | None |
| GET | /api/heatmap | Category × signal frequency heatmap | None |
| POST | /api/override | Admin override (unauthenticated — known V3 gap) | ⚠️ None |
| GET | /api/dataset/files | List downloadable dataset files | None |
| GET | /api/dataset/download/{key} | Download CSV or JSONL dataset | None |

---

## Frontend Components Map

| Component | Tab | Data Source | What It Shows |
|-----------|-----|------------|---------------|
| DecisionBanner | Top bar | latestResult | Full-width color bar — BLOCK/SUPPORT/ALLOW |
| RiskBar | Chat bubble | result.risk_score | Inline 0–100 visual bar per message |
| ExplainPanel | Chat bubble | result.decision_trace | Expandable causal trace in message |
| RiskGauge | Risk tab | result.risk_score | Animated circular risk dial |
| DecisionComparator | Risk tab | result.decision_trace.comparator | BLOCK score vs ALLOW score side-by-side |
| SessionPanel | Risk tab | result.session | Turn count, cumulative risk, distress, escalation |
| RiskGraph | Risk tab | result.session.risk_history | Line chart — risk + distress over turns |
| SignalBreakdown | Evidence tab | result.signal_breakdown | Semantic + session + policy weight bars |
| EvidencePanel | Evidence tab | result.evidence_spans | FAISS nearest-neighbor phrase matches |
| Timeline | Evidence tab | result.timeline | Per-stage latency waterfall |
| MetricsPanel | Monitor tab | GET /api/metrics | Total requests, errors, avg latency |
| HeatmapPanel | Monitor tab | GET /api/heatmap | Category × sub-signal frequency grid |
| AuditLogs | Monitor tab | GET /api/audit | Last 50 MongoDB decisions, live-refresh |

---

## Environment Variables Required

```bash
# Backend (.env)
MONGO_URL=mongodb+srv://...        # MongoDB Atlas or local
DB_NAME=aegis_production
GROQ_API_KEY=gsk_...               # Groq API key
REDIS_URL=redis://...              # Optional — falls back to in-memory
RATE_LIMIT=60/minute               # Configurable rate limit
CORS_ORIGINS=https://your-cdn.cloudfront.net

# Frontend (.env)
VITE_API_BASE=https://api.your-domain.cloudfront.net/api
```

---

*Aegis AI | Chakravyuha V2.5 | Built by Jaswanth | April 2026*
