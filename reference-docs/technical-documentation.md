# Technical Documentation

This document covers service internals, configuration, deployment procedures, API contracts, and operational runbooks for engineers, DevOps, and SREs.

---

## 1. Service Architecture Overview

```mermaid
graph TB
    subgraph Services_and_Ports
        NGINX[NGINX_Port_80]
        Merchant[Merchant_API_Port_8000<br>Python_3_12_FastAPI_SQLModel]
        PSP[PSP_Payment_Port_8001<br>Python_3_12_FastAPI_SQLModel]
        AppsSDK[Apps_SDK_MCP_Port_2091<br>Python_3_12_FastAPI_MCP]
        UI[Next_js_UI_Port_3000<br>Node_18_React_19]
        Promo[Promotion_Agent_Port_8002<br>NAT_Nemotron_3_nano]
        PostPurch[Post_Purchase_Agent_Port_8003<br>NAT_Nemotron_3_nano]
        Reco[Recommendation_Agent_Port_8004<br>NAT_Nemotron_3_nano]
        Search[Search_Agent_Port_8005<br>NAT_Nemotron_3_nano]
    end

    subgraph Infrastructure
        Milvus[Milvus_Standalone_Port_19530]
        MinIO[MinIO_Port_9000]
        Etcd[Etcd_Port_2379]
        Phoenix[Phoenix_Port_6006]
    end
```

---

## 2. Merchant Service Technical Details

### 2.1 Application Entry Point

The FastAPI application is created in `src/merchant/main.py`:
- Registers CORS middleware (configurable origins)
- Adds ACP Headers middleware (Request-Id, idempotency, API-Version)
- Adds Request Logging middleware
- Includes ACP checkout routes at root path
- Includes UCP routes (discovery + A2A) at root path
- Includes shared routes (products, health, metrics) at root path
- Seeds the database with product catalog data on startup
- Creates database tables on startup

### 2.2 API Endpoint Reference

#### ACP Protocol Endpoints

| Method | Path | Auth | Idempotent | Description |
|--------|------|------|------------|-------------|
| POST | `/checkout_sessions` | API Key | Yes | Create checkout session with items |
| GET | `/checkout_sessions/{id}` | API Key | N/A | Retrieve session by ID |
| POST | `/checkout_sessions/{id}` | API Key | Yes | Update session (items, address, shipping) |
| POST | `/checkout_sessions/{id}/complete` | API Key | Yes | Complete checkout with payment |
| POST | `/checkout_sessions/{id}/cancel` | API Key | Yes | Cancel session |

#### UCP Protocol Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/.well-known/ucp` | None | UCP business profile discovery |
| GET | `/.well-known/agent-card.json` | None | A2A agent card for SDK transport |
| POST | `/a2a` | API Key | JSON-RPC 2.0 A2A checkout operations |

#### Shared Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/health` | None | Health check |
| GET | `/products` | API Key | List products (paginated) |
| GET | `/products/{id}` | API Key | Get product by ID |
| POST | `/metrics/agent-outcomes` | API Key | Record agent invocation outcome |
| POST | `/metrics/recommendation-attribution` | API Key | Record recommendation attribution event |

### 2.3 Request Headers

| Header | Required | Purpose | Example |
|--------|----------|---------|---------|
| `Authorization` | Yes (except health, discovery) | Bearer API key | `Bearer merchant-api-key-12345` |
| `API-Version` | Recommended | Protocol version | `2026-01-16` |
| `Idempotency-Key` | Recommended for POST | Deduplication | UUID v4 |
| `Request-Id` | Auto-generated if missing | Request tracing | UUID v4 |
| `Accept-Language` | Optional | Locale preference | `en-US` |

### 2.4 Promotion Agent Integration

The promotion service (`src/merchant/services/promotion.py`) communicates with the NAT agent via an async HTTP client:

```mermaid
sequenceDiagram
    participant Checkout as Checkout_Operations
    participant PromSvc as Promotion_Service
    participant DB as Database
    participant Client as Promotion_Agent_Client
    participant Agent as NAT_Promotion_Agent

    Checkout->>PromSvc: get_promotion_for_line_item
    PromSvc->>DB: query_product_with_competitor_prices
    PromSvc->>PromSvc: compute_inventory_pressure
    PromSvc->>PromSvc: compute_competition_position
    PromSvc->>PromSvc: compute_seasonal_urgency
    PromSvc->>PromSvc: filter_margin_safe_actions
    PromSvc->>Client: send_PromotionContextInput
    Client->>Agent: HTTP_POST_with_10s_timeout
    alt Agent responds
        Agent-->>Client: PromotionDecisionOutput
        Client-->>PromSvc: decision
        PromSvc->>PromSvc: validate_action_in_allowed_list
        PromSvc->>PromSvc: calculate_discount_and_recalculate_totals
    else Timeout or error
        Client-->>PromSvc: None
        PromSvc->>PromSvc: apply_NO_PROMO_fallback
    end
    PromSvc-->>Checkout: enriched_line_item_with_promotion
```

**Timeout**: 10 seconds per agent call. On timeout, the system applies no discount (fail-closed).

### 2.5 Post-Purchase Webhook Dispatch

When a checkout is completed, the merchant:
1. Calls the Post-Purchase Agent to generate a localized message
2. Constructs a webhook payload with order details and the generated message
3. POSTs the webhook to the configured `WEBHOOK_URL`
4. Includes the webhook in the checkout session response

### 2.6 Database Session Management

- SQLAlchemy async engine with SQLite backend
- Session-per-request pattern via FastAPI dependency injection
- All JSON fields stored as serialized strings (`line_items_json`, `buyer_json`, etc.)
- Database created at `./agentic_commerce.db` (configurable via `DATABASE_URL`)

---

## 3. Payment Service Provider (PSP) Technical Details

### 3.1 API Endpoint Reference

| Method | Path | Auth | Idempotent | Description |
|--------|------|------|------------|-------------|
| POST | `/agentic_commerce/delegate_payment` | PSP API Key | Yes | Create vault token |
| POST | `/agentic_commerce/create_and_process_payment_intent` | PSP API Key | Yes | Process payment |
| GET | `/health` | None | N/A | Health check |

### 3.2 Vault Token Lifecycle

```mermaid
stateDiagram_v2
    state "Token Created" as created
    state "Token Active" as active
    state "Token Consumed" as consumed

    created --> active : stored_in_database
    active --> consumed : payment_intent_processed
    active --> active : idempotent_retry
```

**Vault Token Fields**:
- `id`: Prefixed with `vt_` (e.g., `vt_a1b2c3d4e5f6`)
- `payment_method_json`: Encrypted card details (type, number, exp, etc.)
- `allowance_json`: Amount limit tied to specific checkout session
- `status`: `active` (available) or `consumed` (used for payment)

### 3.3 Payment Intent Processing

1. Receive vault token ID and amount
2. Look up vault token in database
3. Verify token status is `active`
4. Verify payment amount matches token allowance (or is less)
5. Mark token as `consumed`
6. Create PaymentIntent record with status `completed`
7. Return payment confirmation

### 3.4 Idempotency Implementation

Both merchant and PSP implement request deduplication:
- An `Idempotency-Key` header is matched against a SHA-256 hash of `method:path:body`
- If the same key + same body hash is received, the cached response is returned
- If the same key + different body hash is received, a `409 Conflict` is returned
- Cache entries expire after 24 hours

---

## 4. Apps SDK (MCP Server) Technical Details

### 4.1 MCP Tool Contracts

#### search_products

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Natural language search query |
| `category` | string | No | Category filter |
| `limit` | integer | No | Max results (default: 3, max: 50) |

**Returns**: Array of matching products with metadata.

#### get_recommendations

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `product_id` | string | Yes | Current product ID |
| `product_name` | string | Yes | Current product name |
| `cart_items` | array | No | Current cart contents |
| `session_id` | string | No | Session ID for attribution tracking |

**Returns**: Three ranked recommendations with reasoning, pipeline trace, and request ID.

#### checkout

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `cart_id` | string | Yes | Cart identifier |
| `buyer` | object | Yes | Buyer name and email |
| `shipping_address` | object | Yes | Delivery address |
| `payment_method` | object | Yes | Card details |

**Returns**: Order confirmation with success status, order ID, total, and message.

### 4.2 SSE Event Stream

**Endpoint**: `GET /events`  
**Content-Type**: `text/event-stream`

**Event Format**:
```
event: checkout_event
data: {"type": "session_create", "endpoint": "/checkout_sessions", "method": "POST", "status": 201, "summary": "...", "sessionId": "..."}

event: agent_activity_event
data: {"agentType": "promotion", "productId": "prod_1", "action": "DISCOUNT_10_PCT", "discountAmount": 250, "reasoning": "...", "signals": {...}}
```

### 4.3 Full Checkout Orchestration

The `process_acp_checkout` function orchestrates the complete ACP checkout flow:

```mermaid
flowchart TD
    Start[Receive_checkout_request] --> ClearEvents[Clear_SSE_event_history]
    ClearEvents --> CreateSession[POST_merchant_checkout_sessions<br>with_cart_items]
    CreateSession --> EmitCreateEvt[Emit_session_create_event]
    EmitCreateEvt --> SelectFulfillment[POST_merchant_checkout_sessions_id<br>with_fulfillment_option_id]
    SelectFulfillment --> EmitUpdateEvt[Emit_session_update_event]
    EmitUpdateEvt --> DelegatePayment[POST_psp_delegate_payment<br>with_card_and_allowance]
    DelegatePayment --> EmitDelegateEvt[Emit_payment_delegation_event]
    EmitDelegateEvt --> Complete[POST_merchant_checkout_sessions_id_complete<br>with_vault_token]
    Complete --> EmitCompleteEvt[Emit_session_complete_event]
    EmitCompleteEvt --> Return[Return_CheckoutOutput]

    CreateSession -->|Error| Fallback[Simulated_checkout_fallback]
    DelegatePayment -->|Error| Fallback
    Complete -->|Error| Fallback
```

### 4.4 Attribution Tracking

The Apps SDK records metrics to the Merchant API (best-effort, non-blocking):

| Event | Endpoint | Trigger |
|-------|----------|---------|
| Agent outcome (recommendation) | `POST /metrics/agent-outcomes` | After every recommendation request |
| Agent outcome (search) | `POST /metrics/agent-outcomes` | After every search request |
| Recommendation impression | `POST /metrics/recommendation-attribution` | When recommendations are returned |
| Recommendation click | `POST /metrics/recommendation-attribution` | When user clicks a recommendation |
| Purchase attribution | `POST /metrics/recommendation-attribution` | When checkout completes with recommended items |

---

## 5. AI Agent Configuration

### 5.1 NAT Agent Common Configuration

All agents share:
- **Runtime**: NeMo Agent Toolkit (`nat serve`)
- **Model**: `nvidia/nemotron-3-nano-30b-a3b` via NVIDIA NIM
- **Transport**: HTTP (one agent per port)
- **Health check**: `GET /health` returns `{"status": "ok"}`

### 5.2 Promotion Agent Config

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Config file | `configs/promotion.yml` | YAML workflow definition |
| Port | 8002 | HTTP service port |
| Temperature | 0.1 | Near-deterministic output |
| Workflow | `chat_completion` | Single LLM call, no tools |
| Input format | `PromotionContextInput` JSON | Signals and allowed actions |
| Output format | `PromotionDecisionOutput` JSON | Action, reason codes, reasoning |

### 5.3 Post-Purchase Agent Config

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Config file | `configs/post-purchase.yml` | YAML workflow definition |
| Port | 8003 | HTTP service port |
| Temperature | 0.3 | Slightly creative for natural language |
| Workflow | `chat_completion` | Single LLM call |
| Input format | Brand persona + order + status | Message generation context |
| Output format | Generated message string | Localized order status message |

### 5.4 Recommendation Agent Config

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Config file | `configs/recommendation.yml` | YAML workflow definition |
| Port | 8004 | HTTP service port |
| Temperature | 0.1 | Near-deterministic recommendations |
| Workflow | ARAG (multi-step) | RAG + parallel agents + ranking |
| Custom components | `parallel_execution`, `rag_retriever`, `output_contract_guard` | Registered in `register.py` |
| Vector DB | Milvus (port 19530) | Product embedding storage |
| Embedding model | `nv-embedqa-e5-v5` | Text-to-vector conversion |

### 5.5 Search Agent Config

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Config file | `configs/search.yml` | YAML workflow definition |
| Port | 8005 | HTTP service port |
| Temperature | 0.0 | Fully deterministic |
| Workflow | RAG-only | Vector retrieval without LLM ranking |
| Custom components | `rag_retriever` | Milvus integration |

---

## 6. Environment Configuration

### 6.1 Required Environment Variables

| Variable | Default | Service(s) | Required |
|----------|---------|-----------|----------|
| `NVIDIA_API_KEY` | — | All agents | **Yes** (for NIM cloud) |
| `MERCHANT_API_KEY` | `merchant-api-key-12345` | Merchant, UI | No |
| `PSP_API_KEY` | `psp-api-key-12345` | PSP, Merchant | No |
| `DATABASE_URL` | `sqlite:///./agentic_commerce.db` | Merchant, PSP | No |

### 6.2 Service URL Variables

| Variable | Default | Used By |
|----------|---------|---------|
| `PROMOTION_AGENT_URL` | `http://localhost:8002` | Merchant |
| `POST_PURCHASE_AGENT_URL` | `http://localhost:8003` | Merchant |
| `RECOMMENDATION_AGENT_URL` | `http://localhost:8004` | Apps SDK |
| `SEARCH_AGENT_URL` | `http://localhost:8005` | Apps SDK |
| `MERCHANT_API_URL` | `http://localhost:8000` | Apps SDK, UI |
| `PSP_API_URL` | `http://localhost:8001` | UI |
| `WEBHOOK_URL` | `http://localhost:3000/api/webhooks/acp` | Merchant |
| `WEBHOOK_SECRET` | `whsec_demo_secret` | Merchant, UI |

### 6.3 NIM Configuration

| Variable | Default | For Local NIM |
|----------|---------|---------------|
| `NIM_LLM_BASE_URL` | `https://integrate.api.nvidia.com/v1` | `http://localhost:8000/v1` |
| `NIM_LLM_MODEL_NAME` | `nvidia/nemotron-3-nano-30b-a3b` | `nvidia/nemotron-3-nano` |
| `NIM_EMBED_BASE_URL` | `https://integrate.api.nvidia.com/v1` | `http://localhost:8001/v1` |
| `NIM_EMBED_MODEL_NAME` | `nvidia/nv-embedqa-e5-v5` | `nvidia/nv-embedqa-e5-v5` |

### 6.4 Infrastructure Variables

| Variable | Default | Usage |
|----------|---------|-------|
| `MILVUS_URI` | `http://localhost:19530` | Vector DB connection |
| `PHOENIX_ENDPOINT` | `http://localhost:6006/v1/traces` | OTLP trace export |

---

## 7. Deployment

### 7.1 Docker Compose Architecture

```mermaid
graph TB
    subgraph docker_compose_yml
        nginx[nginx]
        merchant[merchant]
        psp[psp]
        apps_sdk[apps_sdk]
        ui[ui]
        promo_agent[promotion_agent]
        post_purchase_agent[post_purchase_agent]
        reco_agent[recommendation_agent]
        search_agent[search_agent]
    end

    subgraph docker_compose_infra_yml
        milvus_etcd[milvus_etcd]
        milvus_minio[milvus_minio]
        milvus_standalone[milvus_standalone]
        phoenix[phoenix]
    end

    subgraph docker_compose_nim_yml
        nim_llm[Local_NIM_LLM]
        nim_embed[Local_NIM_Embeddings]
    end

    subgraph Volumes
        acp_data[acp_data_volume]
        milvus_data[milvus_data_volume]
        minio_data[minio_data_volume]
        etcd_data[etcd_data_volume]
    end

    merchant --> acp_data
    psp --> acp_data
    milvus_standalone --> milvus_data
    milvus_minio --> minio_data
    milvus_etcd --> etcd_data
```

### 7.2 Deployment Commands

**Full Docker deployment**:
```bash
# Start infrastructure first
docker compose -f docker-compose.infra.yml up -d

# Wait for Milvus to be healthy
curl -s http://localhost:9091/healthz

# Start application services
docker compose -f docker-compose.infra.yml -f docker-compose.yml up -d

# Optional: Add local NIM
docker compose -f docker-compose.infra.yml -f docker-compose.yml -f docker-compose-nim.yml up -d
```

**One-command setup** (local development):
```bash
./install.sh
```

**Stop all services**:
```bash
./stop.sh
```

### 7.3 Local Development (Without Docker)

```bash
# Backend services
uvicorn src.merchant.main:app --reload                    # Port 8000
uvicorn src.payment.main:app --reload --port 8001         # Port 8001
uvicorn src.apps_sdk.main:app --reload --port 2091        # Port 2091

# Frontend
cd src/ui && pnpm install && pnpm dev                     # Port 3000

# Agents
cd src/agents && uv pip install -e ".[dev]"
nat serve --config_file configs/promotion.yml --port 8002
nat serve --config_file configs/post-purchase.yml --port 8003
nat serve --config_file configs/recommendation.yml --port 8004
nat serve --config_file configs/search.yml --port 8005
```

---

## 8. Health Checks and Monitoring

### 8.1 Service Health Endpoints

| Service | URL | Expected Response |
|---------|-----|-------------------|
| Merchant | `http://localhost:8000/health` | `{"status": "ok"}` |
| PSP | `http://localhost:8001/health` | `{"status": "ok"}` |
| Apps SDK | `http://localhost:2091/health` | `{"status": "ok"}` |
| Promotion Agent | `http://localhost:8002/health` | `{"status": "ok"}` |
| Post-Purchase Agent | `http://localhost:8003/health` | `{"status": "ok"}` |
| Recommendation Agent | `http://localhost:8004/health` | `{"status": "ok"}` |
| Search Agent | `http://localhost:8005/health` | `{"status": "ok"}` |
| Milvus | `http://localhost:9091/healthz` | `{"ok":true}` |

### 8.2 Docker Health Check (Internal)

```bash
docker compose -f docker-compose.infra.yml -f docker-compose.yml exec merchant \
  python -c "import urllib.request as u; \
    print('promotion', u.urlopen('http://promotion-agent:8002/health', timeout=5).status); \
    print('post-purchase', u.urlopen('http://post-purchase-agent:8003/health', timeout=5).status); \
    print('recommendation', u.urlopen('http://recommendation-agent:8004/health', timeout=5).status); \
    print('search', u.urlopen('http://search-agent:8005/health', timeout=5).status)"
```

### 8.3 Log Inspection

```bash
# View specific service logs
docker compose -f docker-compose.infra.yml -f docker-compose.yml logs --tail 200 merchant
docker compose -f docker-compose.infra.yml -f docker-compose.yml logs --tail 200 psp
docker compose -f docker-compose.infra.yml -f docker-compose.yml logs --tail 200 promotion-agent
docker compose -f docker-compose.infra.yml -f docker-compose.yml logs --tail 200 recommendation-agent

# Follow logs in real time
docker compose -f docker-compose.infra.yml -f docker-compose.yml logs -f merchant
```

### 8.4 Phoenix Observability

- **Dashboard**: `http://localhost:6006`
- **OTLP endpoint**: `http://localhost:4317`
- **Traces**: All agent LLM calls are automatically traced
- **Spans**: View prompt input, completion output, latency, token counts

---

## 9. Quality Gates

### 9.1 Backend Checks

```bash
# Type checking
pyright src/

# Linting
ruff check src/ tests/

# Code formatting
ruff format --check src/ tests/

# Unit tests
pytest tests/ -v

# Agent config validation
cd src/agents
nat validate --config_file configs/promotion.yml
nat validate --config_file configs/recommendation.yml
nat validate --config_file configs/post-purchase.yml
nat validate --config_file configs/search.yml
```

### 9.2 Frontend Checks

```bash
cd src/ui/

# TypeScript type checking
pnpm typecheck

# ESLint
pnpm lint

# Prettier format check
pnpm format:check

# Unit tests
pnpm test:run
```

### 9.3 Integration Test Template

```bash
# Create checkout session
curl -s -w "\nHTTP_CODE:%{http_code}" \
  -X POST http://localhost:8000/checkout_sessions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer merchant-api-key-12345" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"items": [{"id": "prod_1", "quantity": 1}]}'

# Delegate payment
curl -s -w "\nHTTP_CODE:%{http_code}" \
  -X POST http://localhost:8001/agentic_commerce/delegate_payment \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer psp-api-key-12345" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"payment_method": {"type": "card", "card_number_type": "fpan", "virtual": false, "number": "4111111111111111", "exp_month": "12", "exp_year": "2027", "display_card_funding_type": "credit", "display_last4": "1111"}, "allowance": {"reason": "one_time", "amount_cents": 2500, "currency": "USD", "checkout_session_id": "SESSION_ID"}}'
```

---

## 10. Troubleshooting Guide

### 10.1 Common Issues

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| 401 Unauthorized | Wrong API key | Check `MERCHANT_API_KEY` or `PSP_API_KEY` env var |
| Promotion agent timeout | NIM endpoint unreachable | Verify `NVIDIA_API_KEY` and `NIM_LLM_BASE_URL` |
| Empty recommendations | Milvus not seeded | Run the vector seeding script via `install.sh` |
| SSE events not appearing | Apps SDK not running | Check `http://localhost:2091/health` |
| Checkout fails at payment | PSP service down | Check `http://localhost:8001/health` |
| UCP discovery returns 404 | Wrong URL path | Use `/.well-known/ucp` (not `/ucp`) |
| 409 Conflict on POST | Reused idempotency key | Generate a new UUID for the Idempotency-Key header |
| Cart lost after restart | In-memory storage | Expected behavior; cart resets with Apps SDK restart |

### 10.2 Architecture Verification Order

When debugging issues:
1. **What does documentation say should happen?** — Check specs in `docs/specs/`
2. **Does the implementation match?** — Read relevant source code
3. **Is documentation outdated or code wrong?** — Cross-reference with tests
4. **Environment or configuration issue?** — Check env vars and health endpoints

### 10.3 Network Troubleshooting (Docker)

```bash
# Verify service discovery within Docker network
docker compose -f docker-compose.infra.yml -f docker-compose.yml exec merchant \
  python -c "import socket; print(socket.getaddrinfo('promotion-agent', 8002))"

# Test inter-service connectivity
docker compose -f docker-compose.infra.yml -f docker-compose.yml exec apps-sdk \
  python -c "import urllib.request as u; print(u.urlopen('http://merchant:8000/health', timeout=5).read())"
```

---

## 11. Dependency Map

```mermaid
graph TD
    subgraph Runtime_Dependencies
        UI -->|requires| Merchant
        UI -->|requires| PSP
        UI -->|requires| AppsSDK[Apps_SDK]
        AppsSDK -->|requires| Merchant
        AppsSDK -->|requires| PSP
        AppsSDK -->|requires| RecoAgent[Recommendation_Agent]
        AppsSDK -->|requires| SearchAgent[Search_Agent]
        Merchant -->|requires| PromoAgent[Promotion_Agent]
        Merchant -->|requires| PostPurchAgent[Post_Purchase_Agent]
        Merchant -->|requires| PSP
        RecoAgent -->|requires| Milvus
        SearchAgent -->|requires| Milvus
        Milvus -->|requires| MinIO
        Milvus -->|requires| Etcd
        PromoAgent -->|requires| NIM[NVIDIA_NIM]
        PostPurchAgent -->|requires| NIM
        RecoAgent -->|requires| NIM
        SearchAgent -->|requires| NIM
    end

    subgraph Optional_Dependencies
        PromoAgent -.->|optional| Phoenix
        PostPurchAgent -.->|optional| Phoenix
        RecoAgent -.->|optional| Phoenix
        SearchAgent -.->|optional| Phoenix
    end
```

**Startup Order** (recommended):
1. Infrastructure: Etcd, MinIO, Milvus, Phoenix
2. Core services: Merchant, PSP
3. Agents: Promotion, Post-Purchase, Recommendation, Search
4. Integration: Apps SDK
5. Frontend: UI, NGINX
