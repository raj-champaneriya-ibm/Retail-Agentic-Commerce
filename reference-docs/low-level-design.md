# Low-Level Design (LLD)

## 1. Component Architecture

This document details the internal design of each service, including data models, API contracts, algorithms, and inter-service communication patterns.

---

## 2. Merchant Service

### 2.1 Module Structure

```mermaid
graph TB
    subgraph Merchant_Service
        Main[main_py_FastAPI_App]
        subgraph API_Layer
            SharedRoutes[shared_routes_products_health_metrics]
            ACPRoutes[acp_checkout_routes]
            UCPRoutes[ucp_discovery_and_a2a_routes]
        end
        subgraph Middleware
            ACPHeaders[ACP_Headers_Middleware]
            RequestLog[Request_Logging_Middleware]
        end
        subgraph Domain
            CheckoutModels[checkout_domain_models]
            CheckoutOps[checkout_operations]
        end
        subgraph Services
            PromotionSvc[promotion_service]
            PostPurchaseSvc[post_purchase_service]
            IdempotencySvc[idempotency_service]
            AgentOutcomesSvc[agent_outcomes_service]
            RecoAttrSvc[recommendation_attribution_service]
        end
        subgraph Database
            Models[db_models]
            Session[db_session_manager]
        end
    end

    Main --> ACPHeaders
    Main --> RequestLog
    Main --> SharedRoutes
    Main --> ACPRoutes
    Main --> UCPRoutes
    ACPRoutes --> CheckoutOps
    UCPRoutes --> CheckoutOps
    CheckoutOps --> PromotionSvc
    CheckoutOps --> PostPurchaseSvc
    CheckoutOps --> CheckoutModels
    SharedRoutes --> AgentOutcomesSvc
    SharedRoutes --> RecoAttrSvc
    PromotionSvc --> Models
    CheckoutOps --> Models
    Models --> Session
```

### 2.2 Database Entity Relationship

```mermaid
erDiagram
    Customer ||--o{ BrowseHistory : has
    Product ||--o{ BrowseHistory : viewed_in
    Product ||--o{ CompetitorPrice : has
    CheckoutSession ||--o{ AgentInvocationOutcome : tracked_by
    CheckoutSession ||--o{ RecommendationAttributionEvent : attributed_to

    Customer {
        string id PK
        string email UK
        string name
        datetime created_at
    }

    Product {
        string id PK
        string sku UK
        string name
        int base_price
        int stock_count
        float min_margin
        string image_url
        string lifecycle
        string demand_velocity
    }

    CheckoutSession {
        string id PK
        string protocol
        string status
        string currency
        string locale
        string line_items_json
        string buyer_json
        string fulfillment_address_json
        string fulfillment_options_json
        string selected_fulfillment_option_id
        string totals_json
        string order_json
        string messages_json
        string links_json
        string metadata_json
        datetime created_at
        datetime updated_at
        datetime expires_at
    }

    BrowseHistory {
        int id PK
        string customer_id FK
        string category
        string search_term
        string product_id FK
        int price_viewed
        datetime viewed_at
    }

    CompetitorPrice {
        int id PK
        string product_id FK
        string retailer_name
        int price
        datetime updated_at
    }

    AgentInvocationOutcome {
        int id PK
        datetime timestamp
        string agent_type
        string channel
        string status
        int latency_ms
        string request_id
        string session_id
        string error_code
    }

    RecommendationAttributionEvent {
        int id PK
        datetime timestamp
        string event_type
        string session_id
        string recommendation_request_id
        string product_id
        int position
        string order_id
        int quantity
        int revenue_cents
        string source
    }
```

### 2.3 ACP Checkout Route Contract

#### Create Session

| Field | Type | Direction | Description |
|-------|------|-----------|-------------|
| `items` | Array of `{id, quantity}` | Request | Products to purchase |
| `buyer` | Object `{first_name, last_name, email, phone_number}` | Request (optional) | Buyer identity |
| `fulfillment_address` | Object | Request (optional) | Shipping address |
| `metadata` | Object `{discounts: {codes: []}}` | Request (optional) | Discount codes |
| `id` | String | Response | Session identifier (e.g., `checkout_abc123`) |
| `status` | Enum | Response | `not_ready_for_payment` initially |
| `line_items` | Array of LineItem | Response | Items with promotion data applied |
| `fulfillment_options` | Array of FulfillmentOption | Response | Available shipping methods |
| `totals` | Array of Total | Response | Breakdown: subtotal, discount, tax, fulfillment, total |

#### Update Session

| Field | Type | Direction | Description |
|-------|------|-----------|-------------|
| `items` | Array | Request (optional) | Replace cart items |
| `fulfillment_option_id` | String | Request (optional) | Selected shipping method |
| `fulfillment_address` | Object | Request (optional) | Shipping destination |
| `discounts` | Object | Request (optional) | Discount code updates |

Fulfillment address and shipping selection cause status to transition from `not_ready_for_payment` to `ready_for_payment`.

#### Complete Session

| Field | Type | Direction | Description |
|-------|------|-----------|-------------|
| `payment_data` | Object `{token, provider, ...}` | Request | Vault token from PSP delegation |
| `status` | `completed` | Response | Final state |
| `order` | Object `{id, completed_at, message}` | Response | Order confirmation details |

### 2.4 Promotion Service Algorithm

```mermaid
flowchart TD
    Start[Receive_Line_Item] --> QueryDB[Query_Product_From_Database]
    QueryDB --> ComputeSignals[Compute_Business_Signals]

    ComputeSignals --> InvPressure{Stock_Count_Greater_50}
    InvPressure -->|Yes| HighInv[inventory_pressure_HIGH]
    InvPressure -->|No| LowInv[inventory_pressure_LOW]

    ComputeSignals --> CompPrice{Compare_With_Competitors}
    CompPrice -->|Higher| AboveMkt[competition_ABOVE_MARKET]
    CompPrice -->|Equal| AtMkt[competition_AT_MARKET]
    CompPrice -->|Lower| BelowMkt[competition_BELOW_MARKET]

    ComputeSignals --> Season{Check_Retail_Calendar}
    Season -->|Within_3_days| Peak[seasonal_PEAK]
    Season -->|Before_event| PreSeason[seasonal_PRE_SEASON]
    Season -->|After_event| PostSeason[seasonal_POST_SEASON]
    Season -->|None| OffSeason[seasonal_OFF_SEASON]

    HighInv --> FilterActions[Filter_Margin_Safe_Actions]
    LowInv --> FilterActions
    AboveMkt --> FilterActions
    AtMkt --> FilterActions
    BelowMkt --> FilterActions
    Peak --> FilterActions
    PreSeason --> FilterActions
    PostSeason --> FilterActions
    OffSeason --> FilterActions

    FilterActions --> MarginCheck{base_price_times_1_minus_discount_gte_base_price_times_min_margin}
    MarginCheck -->|Pass| AllowedActions[Build_Allowed_Action_List]
    MarginCheck -->|Fail| ExcludeAction[Exclude_This_Action]
    ExcludeAction --> MarginCheck

    AllowedActions --> CallAgent[Send_Context_to_NAT_Promotion_Agent]
    CallAgent --> AgentDecision{Agent_Returns_Decision}
    AgentDecision -->|Valid_action_in_allowed_list| ApplyDiscount[Calculate_Discount_Amount]
    AgentDecision -->|Invalid_or_timeout| Fallback[Fallback_NO_PROMO]

    ApplyDiscount --> UpdateLineItem[Update_LineItem_discount_subtotal_tax_total]
    Fallback --> UpdateLineItem
    UpdateLineItem --> Return[Return_Enriched_Session]
```

**Discount Map** (action to percentage):

| Action | Discount Rate |
|--------|--------------|
| `NO_PROMO` | 0% |
| `DISCOUNT_5_PCT` | 5% |
| `DISCOUNT_10_PCT` | 10% |
| `DISCOUNT_15_PCT` | 15% |
| `FREE_SHIPPING` | 0% (shipping cost waived) |

### 2.5 Checkout Domain Models

#### LineItem Structure

```mermaid
classDiagram
    class LineItem {
        +String id
        +ItemRef item
        +String name
        +int base_amount
        +int discount
        +int subtotal
        +int tax
        +int total
        +PromotionMetadata promotion
    }

    class ItemRef {
        +String id
        +int quantity
    }

    class PromotionMetadata {
        +String action
        +List reason_codes
        +String reasoning
        +int stock_count
        +PromotionSignals signals
    }

    class PromotionSignals {
        +String inventory_pressure
        +String competition_position
        +String seasonal_urgency
        +String product_lifecycle
        +String demand_velocity
    }

    class FulfillmentOption {
        +String type
        +String id
        +String title
        +String subtitle
        +int subtotal
        +int tax
        +int total
        +String carrier_info
        +String earliest_delivery_time
        +String latest_delivery_time
    }

    class Total {
        +String type
        +String display_text
        +int amount
    }

    LineItem --> ItemRef
    LineItem --> PromotionMetadata
    PromotionMetadata --> PromotionSignals
```

### 2.6 UCP A2A JSON-RPC Methods

| Method | Parameters | Response | Description |
|--------|-----------|----------|-------------|
| `a2a.ucp.checkout.create` | `meta`, `checkout` (items, buyer, address) | `contextId`, `parts` (checkout data, UCP metadata) | Create a new UCP checkout |
| `a2a.ucp.checkout.get` | `meta`, `contextId` | `parts` (current checkout state) | Retrieve UCP session |
| `a2a.ucp.checkout.update` | `meta`, `contextId`, `checkout` | `parts` (updated state) | Update fulfillment, address |
| `a2a.ucp.checkout.complete` | `meta`, `contextId`, `payment_data` | `parts` (completed order) | Process payment and complete |
| `a2a.ucp.checkout.cancel` | `meta`, `contextId` | `parts` (canceled state) | Cancel the session |

### 2.7 UCP Discovery Response Structure

The `/.well-known/ucp` endpoint returns:

| Field | Type | Content |
|-------|------|---------|
| `name` | String | Merchant display name |
| `url` | String | Base URL |
| `description` | String | Business description |
| `capabilities` | Array | Supported UCP capabilities with versions |
| `services` | Array | Offered services (checkout, fulfillment, discounts) |
| `payment_handlers` | Array | Accepted payment methods |
| `signing_keys` | Array | JWK keys for webhook verification |

---

## 3. Payment Service Provider (PSP)

### 3.1 Module Structure

```mermaid
graph TB
    subgraph PSP_Service
        Main[main_py_FastAPI_App]
        subgraph Routes
            Delegate[delegate_payment_route]
            Process[create_and_process_payment_intent_route]
            Health[health_check]
        end
        subgraph Services
            VaultSvc[vault_token_service]
            PaymentSvc[payment_intent_service]
            IdempSvc[idempotency_service]
        end
        subgraph Database
            VaultModel[VaultToken_model]
            IntentModel[PaymentIntent_model]
            IdempModel[IdempotencyRecord_model]
        end
    end

    Main --> Delegate
    Main --> Process
    Delegate --> VaultSvc
    Process --> PaymentSvc
    VaultSvc --> VaultModel
    PaymentSvc --> IntentModel
    PaymentSvc --> VaultModel
    Delegate --> IdempSvc
    Process --> IdempSvc
    IdempSvc --> IdempModel
```

### 3.2 Payment Delegation Flow

```mermaid
sequenceDiagram
    participant Client as Client_Agent
    participant PSP as PSP_API
    participant DB as PSP_Database

    Client->>PSP: POST /agentic_commerce/delegate_payment
    Note right of Client: payment_method, allowance, billing_address

    PSP->>PSP: Validate API key
    PSP->>PSP: Check idempotency key
    PSP->>DB: Create VaultToken record
    Note right of DB: status = active
    PSP-->>Client: 201 Created
    Note left of PSP: vault_token, idempotency_key_echo, allowance
```

### 3.3 Payment Processing Flow

```mermaid
sequenceDiagram
    participant Merchant as Merchant_API
    participant PSP as PSP_API
    participant DB as PSP_Database

    Merchant->>PSP: POST /agentic_commerce/create_and_process_payment_intent
    Note right of Merchant: vault_token, amount, currency

    PSP->>DB: Look up VaultToken by ID
    alt Token not found or consumed
        PSP-->>Merchant: 400 Bad Request
    else Token active
        PSP->>DB: Update VaultToken status to consumed
        PSP->>DB: Create PaymentIntent record
        Note right of DB: status = completed
        PSP-->>Merchant: 200 OK
        Note left of PSP: id, status, amount, currency
    end
```

### 3.4 Vault Token States

```mermaid
stateDiagram-v2
    active --> consumed : payment_intent_created
    active --> active : duplicate_idempotent_request
```

---

## 4. Apps SDK (MCP Server)

### 4.1 MCP Tool Registry

```mermaid
graph TB
    subgraph MCP_Server
        Transport[Stateless_HTTP_Transport]
        subgraph Tools
            SearchProducts[search_products_tool]
            GetReco[get_recommendations_tool]
            AddCart[add_to_cart_tool]
            RemoveCart[remove_from_cart_tool]
            UpdateCart[update_cart_quantity_tool]
            GetCart[get_cart_tool]
            Checkout[checkout_tool]
            TrackClick[track_recommendation_click_tool]
            CreateSession[create_acp_session_tool]
            UpdateSession[update_acp_session_tool]
        end
        subgraph State
            CartStore[in_memory_cart_store]
            EventBus[SSE_event_emitter]
        end
    end

    subgraph External
        MerchantAPI[Merchant_API]
        PSPAPI[PSP_API]
        RecoAgent[Recommendation_Agent]
        SearchAgent[Search_Agent]
    end

    Transport --> Tools
    SearchProducts --> SearchAgent
    GetReco --> RecoAgent
    AddCart --> CartStore
    RemoveCart --> CartStore
    UpdateCart --> CartStore
    GetCart --> CartStore
    Checkout --> MerchantAPI
    Checkout --> PSPAPI
    CreateSession --> MerchantAPI
    UpdateSession --> MerchantAPI
    TrackClick --> MerchantAPI
    Tools --> EventBus
```

### 4.2 Cart Calculation Logic

| Component | Calculation | Example |
|-----------|-------------|---------|
| Subtotal | Sum of (item.basePrice * item.quantity) for all items | 2 shirts at $25.00 = $50.00 |
| Shipping | Flat rate $5.00 | $5.00 |
| Tax | 8.75% of subtotal | $50.00 * 0.0875 = $4.38 |
| Total | subtotal + shipping + tax | $50.00 + $5.00 + $4.38 = $59.38 |

All monetary values are stored and transmitted in **cents** (integer) to avoid floating-point precision issues.

### 4.3 SSE Event Types

```mermaid
graph LR
    subgraph Event_Sources
        ACPOps[ACP_Session_Operations]
        AgentDecisions[Agent_Promotion_Decisions]
        RecoOps[Recommendation_Operations]
    end

    subgraph Event_Types
        CheckoutEvt[checkout_event]
        AgentEvt[agent_activity_event]
    end

    subgraph Consumers
        BusinessPanel[Business_Panel_UI]
        ActivityPanel[Agent_Activity_Panel_UI]
    end

    ACPOps --> CheckoutEvt
    AgentDecisions --> AgentEvt
    RecoOps --> AgentEvt
    CheckoutEvt --> BusinessPanel
    AgentEvt --> ActivityPanel
```

---

## 5. AI Agents (NAT)

### 5.1 Promotion Agent Decision Tree

```mermaid
flowchart TD
    Input[Receive_PromotionContextInput] --> CheckInv{inventory_pressure}

    CheckInv -->|LOW| NoPromo1[NO_PROMO]
    CheckInv -->|HIGH| CheckComp{competition_position}

    CheckComp -->|BELOW_MARKET| NoPromoOrShip[NO_PROMO_or_FREE_SHIPPING]
    CheckComp -->|AT_MARKET| Disc5[DISCOUNT_5_PCT]
    CheckComp -->|ABOVE_MARKET| Disc10[DISCOUNT_10_PCT]

    NoPromoOrShip --> Modifiers
    Disc5 --> Modifiers
    Disc10 --> Modifiers
    NoPromo1 --> Modifiers

    Modifiers{Apply_Modifiers}
    Modifiers -->|seasonal_PEAK| UpgradeDiscount[Upgrade_One_Level]
    Modifiers -->|lifecycle_CLEARANCE| UpgradeDiscount
    Modifiers -->|demand_DECELERATING| UpgradeDiscount
    Modifiers -->|lifecycle_NEW_ARRIVAL| DowngradeToShipping[Downgrade_to_FREE_SHIPPING]
    Modifiers -->|demand_ACCELERATING| DowngradeToShipping
    Modifiers -->|seasonal_PRE_SEASON| AddShipping[FREE_SHIPPING_if_NO_PROMO]

    UpgradeDiscount --> Output[PromotionDecisionOutput]
    DowngradeToShipping --> Output
    AddShipping --> Output
```

### 5.2 Recommendation Agent ARAG Pipeline

```mermaid
sequenceDiagram
    participant Client as Apps_SDK
    participant RAG as RAG_Retriever
    participant Milvus as Milvus_Vector_DB
    participant NLI as NLI_Agent
    participant UUA as User_Understanding_Agent
    participant Summary as Context_Summary_Agent
    participant Ranker as Item_Ranker
    participant Guard as Output_Contract_Guard

    Client->>RAG: product_context_and_cart
    RAG->>Milvus: vector_similarity_search
    Milvus-->>RAG: candidate_products

    par Parallel Execution
        RAG->>NLI: candidates_for_filtering
        NLI-->>RAG: relevance_scored_items
    and
        RAG->>UUA: user_context_analysis
        UUA-->>RAG: user_intent_signals
    end

    RAG->>Summary: NLI_results_plus_UUA_insights
    Summary-->>RAG: synthesized_context

    RAG->>Ranker: context_plus_candidates
    Ranker-->>RAG: ranked_top_3

    RAG->>Guard: validate_output_schema
    Guard-->>RAG: validated_recommendations

    RAG-->>Client: 3_ranked_recommendations
```

### 5.3 Agent Configuration Summary

| Agent | Model | Temperature | Workflow | Custom Components |
|-------|-------|-------------|----------|-------------------|
| Promotion | nemotron-3-nano-30b | 0.1 | chat_completion | None |
| Post-Purchase | nemotron-3-nano-30b | 0.3 | chat_completion | None |
| Recommendation | nemotron-3-nano-30b | 0.1 | ARAG (multi-step) | parallel_execution, rag_retriever, output_contract_guard |
| Search | nemotron-3-nano-30b | 0.0 | RAG-only | rag_retriever |

---

## 6. Frontend UI

### 6.1 Component Hierarchy

```mermaid
graph TB
    subgraph Page_Layout
        Page[page_tsx_main_layout]
        subgraph Providers
            ACPProv[ACPLogProvider]
            AgentProv[AgentActivityLogProvider]
        end
        subgraph Panels
            AgentPanel[AgentPanel_left]
            BusinessPanel[BusinessPanel_center]
            ActivityPanel[AgentActivityPanel_right]
        end
    end

    subgraph Agent_Panel_Components
        ProductGrid[ProductCard_grid]
        CheckoutModal[CheckoutCard_modal]
        PaymentForm[PaymentShippingForm]
        SearchBar[ProductSearch]
    end

    subgraph Business_Panel_Components
        ACPTab[ACP_Protocol_Log]
        UCPTab[UCP_Protocol_Log]
        SessionState[Session_State_Display]
    end

    subgraph Activity_Panel_Components
        PromoDecisions[Promotion_Decisions]
        RecoDecisions[Recommendation_Results]
        WebhookBridge[Webhook_To_Activity_Bridge]
    end

    Page --> Providers
    Providers --> Panels
    AgentPanel --> ProductGrid
    AgentPanel --> CheckoutModal
    AgentPanel --> PaymentForm
    AgentPanel --> SearchBar
    BusinessPanel --> ACPTab
    BusinessPanel --> UCPTab
    BusinessPanel --> SessionState
    ActivityPanel --> PromoDecisions
    ActivityPanel --> RecoDecisions
    ActivityPanel --> WebhookBridge
```

### 6.2 State Management via Hooks

```mermaid
graph LR
    subgraph Hooks
        useCheckout[useCheckoutFlow]
        useACP[useACPLog]
        useActivity[useAgentActivityLog]
        useEvents[useCheckoutEvents]
        useMCP[useMCPClient]
        useWebhook[useWebhookNotifications]
        useMetrics[useMetrics]
        useTelemetry[usePhoenixTelemetry]
    end

    subgraph Data_Sources
        MerchantAPI[Merchant_REST_API]
        AppsSDKSSE[Apps_SDK_SSE_Stream]
        WebhookEndpoint[Webhook_POST_Endpoint]
    end

    useCheckout --> MerchantAPI
    useEvents --> AppsSDKSSE
    useWebhook --> WebhookEndpoint
    useACP --> useCheckout
    useActivity --> useEvents
    useMCP --> MerchantAPI
```

### 6.3 Protocol Adapter Pattern

The frontend supports dual-protocol operation through an adapter pattern in the API client:

| Operation | ACP (REST) | UCP (A2A JSON-RPC) |
|-----------|-----------|---------------------|
| Create session | `POST /checkout_sessions` | `POST /a2a` method: `a2a.ucp.checkout.create` |
| Get session | `GET /checkout_sessions/{id}` | `POST /a2a` method: `a2a.ucp.checkout.get` |
| Update session | `POST /checkout_sessions/{id}` | `POST /a2a` method: `a2a.ucp.checkout.update` |
| Complete session | `POST /checkout_sessions/{id}/complete` | `POST /a2a` method: `a2a.ucp.checkout.complete` |
| Cancel session | `POST /checkout_sessions/{id}/cancel` | `POST /a2a` method: `a2a.ucp.checkout.cancel` |

---

## 7. Middleware Pipeline

### 7.1 Request Processing Order

```mermaid
sequenceDiagram
    participant Client as HTTP_Client
    participant CORS as CORS_Middleware
    participant Log as Request_Logging_Middleware
    participant ACP as ACP_Headers_Middleware
    participant Auth as API_Key_Dependency
    participant Route as Route_Handler

    Client->>CORS: incoming_request
    CORS->>Log: pass_through
    Log->>ACP: start_timer

    ACP->>ACP: generate_Request_Id
    ACP->>ACP: extract_API_Version
    ACP->>ACP: extract_Accept_Language

    alt POST with Idempotency-Key
        ACP->>ACP: check_idempotency_cache
        alt Cache hit
            ACP-->>Client: return_cached_response
        else Cache miss
            ACP->>Auth: continue_processing
        end
    else GET or no idempotency key
        ACP->>Auth: continue_processing
    end

    Auth->>Auth: verify_Bearer_token
    alt Invalid token
        Auth-->>Client: 401_Unauthorized
    else Valid token
        Auth->>Route: execute_handler
        Route-->>ACP: response
        ACP->>ACP: cache_response_if_2xx
        ACP-->>Log: response_with_Request_Id
        Log->>Log: log_duration_and_status
        Log-->>Client: final_response
    end
```

---

## 8. Error Handling Strategy

| Layer | Error Type | Handling | User Impact |
|-------|-----------|----------|-------------|
| **Middleware** | Missing/invalid API key | 401 Unauthorized | Request rejected |
| **Middleware** | Idempotency conflict | 409 Conflict | Retry with new key |
| **Checkout** | Product not found | 404 Not Found | Item removed from session |
| **Promotion** | Agent timeout (10s) | Fallback to NO_PROMO | No discount applied |
| **Promotion** | Invalid action returned | Reject, use NO_PROMO | No discount applied |
| **Promotion** | Margin violation | Reject action | No discount applied |
| **Payment** | Vault token consumed | 400 Bad Request | Re-delegate payment |
| **Payment** | Token not found | 400 Bad Request | Re-delegate payment |
| **Recommendation** | Agent unreachable | Return empty list | No recommendations shown |
| **SSE** | Connection lost | Auto-reconnect (client) | Brief event gap |
| **UCP** | Escalation required | `requires_escalation` status | Redirect to external URL |
