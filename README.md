# DataTrust — Policy-Aware Enterprise AI Assistant for Secure RAG

An AI orchestration system that enforces role-based LLM access controls, detects shadow AI usage, and gives enterprises audit traceability over AI interactions — built with secure RAG, semantic search, deterministic policy enforcement, guarded prompt construction, and structured JSON API responses.

## The Problem

Enterprises are adopting LLMs faster than they can govern them. Employees paste sensitive data into AI tools with no oversight ("shadow AI"), and there's often no audit trail of what an AI system was allowed to access, or why it returned what it returned. DataTrust addresses this at the infrastructure level rather than trying to police individual employee behavior after the fact.

Most RAG demos assume every user can retrieve from the same document pool, which breaks down in enterprise settings where HR, engineering, admin, and operations data have different confidentiality levels. DataTrust was motivated by that gap: it demonstrates how a RAG assistant can enforce department- and authorization-level access before retrieval, instead of retrieving first and hoping the LLM does not leak sensitive context. It also models “shadow AI” risks, such as employees trying to paste internal documents, source code, or employee data into external AI tools.

## How It Works

1. A user submits a query through the React web app, either through the authenticated chat interface or the local demo chat flow.
2. The backend authenticates the user through an Auth0 Bearer token, with an `X-User-Id` fallback used for demo/development flows, then maps the user to a DataTrust app profile in Supabase.
3. The system resolves the user’s department, authorization level, admin flag, and resource scopes from Supabase.
4. Before retrieval, the query is normalized and evaluated by a deterministic policy engine that detects PII, credentials, prompt injection, out-of-scope access attempts, data exfiltration, admin-scope requests, and external AI/shadow-AI behavior.
5. If the request is blocked, the backend returns a structured policy response explaining the safe reason category and suggested alternative.
6. If the request is allowed, the retrieval orchestrator selects only permitted sources such as Confluence, GitHub, or Google Drive based on query semantics and the user’s allowed scopes.
7. Vector retrieval is enforced in Postgres/pgvector by filtering on active chunks, resource scope IDs, department code, minimum authorization rank, and selected source type before returning candidate context.
8. The backend builds a guarded prompt containing only authorized chunks and sends it to a local Ollama-compatible LLM endpoint.
9. The generated answer is scanned by an output guard for sensitive values such as emails, phone numbers, SSNs, AWS keys, and private-key blocks; unsafe values are redacted before the response is returned.
10. The system logs policy events, sync events, chat history, source references, selected sources, and timing metadata to Supabase and MongoDB-backed storage for traceability.
11. Shadow AI detection works through deterministic signals: mentions of external AI tools combined with transfer actions or sensitive internal targets such as source code, internal documents, customer data, employee data, private repositories, or confidential records.

## Architecture

![Architecture](./docs/architecture.png)

DataTrust is organized as a policy-first RAG pipeline. The frontend sends chat requests to a FastAPI backend, which authenticates the user, resolves their department and authorization level from Supabase, evaluates the query through a deterministic policy engine, and only then performs scoped retrieval from Postgres/pgvector. Retrieved chunks are constrained by source, resource scope, department, and auth rank before being passed into a guarded prompt for the local Ollama-compatible LLM. The final answer is validated by an output guard and returned with policy metadata, selected sources, source references, and timing information.

## Tech Stack

- **Backend:** Python, FastAPI
- **Frontend:** TypeScript, React, Vite
- **LLM Provider:** Local Ollama-compatible LLM endpoint; current generation code uses `phi3:latest` for general/internal answers and `qwen2.5-coder:7b` for code-centric prompts
- **Embeddings:** `sentence-transformers/all-MiniLM-L6-v2`
- **Vector Store:** Postgres with pgvector through Supabase DB connection
- **Metadata / Auth Mapping / Audit:** Supabase tables for users, departments, auth levels, resource scopes, documents, chunks, policy events, and sync events
- **Chat History / Mongo-backed Storage:** MongoDB
- **Auth / Access Control:** Auth0 Bearer tokens mapped to DataTrust users in Supabase; access enforced with department, auth level rank, admin flag, and resource scopes
- **Deployment:** Local development / demo-oriented setup; frontend uses Vite dev proxy to route `/api` calls to the FastAPI backend

## Key Technical Decisions

- **Policy-before-retrieval instead of retrieval-before-filtering:**  
  DataTrust evaluates the user’s prompt and resolves the user’s department, authorization rank, and resource scopes before vector search. This prevents unauthorized documents from entering the LLM context window in the first place, which is safer than relying on the model to ignore sensitive retrieved text.

- **Postgres/pgvector with metadata filters instead of a single unfiltered vector index:**  
  The vector query filters by active chunk status, allowed resource scope IDs, department code, minimum authorization rank, and selected source type. This keeps semantic search useful while enforcing enterprise access-control boundaries at retrieval time.

- **Deterministic policy engine for high-risk checks:**  
  Instead of asking the LLM to decide whether a request is safe, DataTrust uses explicit regexes, keyword rules, department-scope checks, and auth-level checks for PII, secrets, prompt injection, exfiltration, admin-scope requests, and external AI usage. This makes policy behavior easier to test, audit, and explain.

- **Guarded prompt plus output validation:**  
  The model receives only authorized chunks inside a guarded prompt, then the generated answer is scanned for sensitive outputs such as emails, phone numbers, SSNs, AWS keys, and private-key material before it is returned.

- **Structured JSON responses for auditability:**  
  Chat responses include status, policy metadata, selected sources, source references, retrieval count, output-validation status, and timing metadata. This makes responses easier to log, inspect, and debug than free-form text alone.

## Results / Evaluation

Current validation focuses on functional checks and security-path testing during development:

- **Build validation:** The backend Python application compiles successfully, and the React/Vite frontend builds successfully for production.
- **Access-control design validation:** Retrieval is constrained by resource scope, department, authorization rank, active chunk status, and selected source type before context is sent to the LLM.
- **Policy coverage:** The policy layer includes deterministic checks for PII, credentials, prompt injection, data exfiltration, external AI/shadow-AI usage, admin-scope requests, department-scope violations, and authorization-level violations.
- **Output safety:** Generated responses are scanned and redacted for sensitive values such as emails, phone numbers, SSNs, AWS keys, and private-key blocks.

Planned evaluation:
- Create a hand-labeled test set of allowed, blocked, redacted, and out-of-scope prompts.
- Measure policy block accuracy on adversarial access-control prompts.
- Measure retrieval precision on department-specific internal documents.
- Measure average end-to-end latency for normal chat, streaming chat, and blocked requests.

## Screenshots / Demo

Suggested demo assets:

1. **Allowed internal RAG query:** A user asks a scoped question about an authorized internal document and receives an answer with source references.
2. **Blocked policy query:** A user asks for payroll, SSNs, source code exfiltration, or external AI sharing, and DataTrust returns a structured policy denial.
3. **Admin dashboard / audit view:** An admin sees document counts, source distribution, recent documents, recent policy events, connector health, or data-quality status.

## Getting Started

```bash
git clone https://github.com/SJSU-DataTrust/DataTrust_Project
cd DataTrust_Project

# Backend
python -m venv .venv
source .venv/bin/activate
pip install -r backend/requirements.txt

# Create backend/.env and configure at minimum:
# LLM_URL=http://localhost:11434
# OLLAMA_MODEL=phi3
# SUPABASE_URL=...
# SUPABASE_SERVICE_ROLE_KEY=...
# SUPABASE_ANON_KEY=...
# SUPABASE_DB_URL=...
# MONGODB_URI=...
# AUTH0_DOMAIN=...
# AUTH0_AUDIENCE=...
# AUTH0_ISSUER=...

cd backend
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Frontend
cd frontend
npm install
npm run dev

# Useful local endpoints
curl http://localhost:8000/health
# Frontend runs at http://localhost:5173
# Demo route: http://localhost:5173/demo

Future Improvements
Add automated unit and integration tests for policy decisions, retrieval authorization filters, Auth0 mapping, and output redaction.

Gate the X-User-Id demo fallback so it is only available in development or demo mode.

Replace hardcoded model names with environment-driven model configuration.

Add a formal evaluation suite with labeled allowed, blocked, redacted, and out-of-scope prompts.

Add fine-grained document-level and group-level permissions beyond department and auth rank.

Add a /me endpoint so the frontend can use backend-derived admin status instead of hardcoded email checks.

Add Docker Compose and database seed scripts for easier local setup.

Improve observability with structured logs, request IDs, latency metrics, block-rate metrics, and retrieval-quality dashboards.

Add optional reranking and configurable context-window limits for better answer quality.

Expand shadow AI detection with more labeled examples and configurable enterprise policy rules.

Author
Nidhi Shah — linkedin.com/in/nidhishah4065 · github.com/nace129
