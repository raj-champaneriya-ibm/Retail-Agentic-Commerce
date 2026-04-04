# C4 and TOGAF Diagrams

All diagrams in this document use Mermaid.js syntax and follow TOGAF alignment for logical, application, and technology views. Labels use only alphanumeric characters and underscores.

---

## 1. C4 Model Diagrams

### 1.1 Level 1: System Context Diagram

Shows the Retail Agentic Commerce system in relation to its users and external systems.

```mermaid
graph TB
    Consumer[Consumer_User<br>Browses_products_and_initiates_purchases]
    Developer[Developer<br>Configures_agents_and_deploys_services]

    subgraph Retail_Agentic_Commerce_Platform
        System[Agentic_Commerce_System<br>AI_powered_e_commerce_with<br>dual_protocol_checkout]
    end

    NIM_Cloud[NVIDIA_NIM_Cloud<br>LLM_inference_and<br>embedding_endpoints]

    Consumer -->|browses_searches_purchases| System
    Developer -->|configures_deploys_monitors| System
    System -->|LLM_inference_requests| NIM_Cloud
```

### 1.2 Level 2: Container Diagram

Shows the high-level containers (services) within the system.

```mermaid
graph TB
    Consumer[Consumer_User]

    subgraph Retail_Agentic_Commerce_Platform
        subgraph Frontend_Tier
            UI[Next_js_UI<br>React_19_Tailwind_Kaizen<br>Three_panel_layout]
            NGINX[NGINX_Gateway<br>Reverse_proxy_and<br>request_routing]
        end

        subgraph Application_Tier
            Merchant[Merchant_API<br>FastAPI_Python_3_12<br>ACP_and_UCP_protocols]
            PSP[Payment_Service<br>FastAPI_Python_3_12<br>Vault_tokens_and_intents]
            AppsSDK[Apps_SDK_MCP_Server<br>FastAPI_Python_3_12<br>Shopping_tools_and_SSE]
        end

        subgraph Agent_Tier
            Promo[Promotion_Agent<br>NAT_Nemotron_3<br>Dynamic_pricing]
            PostPurch[Post_Purchase_Agent<br>NAT_Nemotron_3<br>Multilingual_messages]
            Reco[Recommendation_Agent<br>NAT_Nemotron_3<br>ARAG_pipeline]
            Search[Search_Agent<br>NAT_Nemotron_3<br>Semantic_search]
        end

        subgraph Data_Tier
            SQLite[(SQLite<br>Checkout_sessions<br>Products_Payments)]
            Milvus[(Milvus<br>Product_vectors<br>Similarity_search)]
        end

        subgraph Observability_Tier
            Phoenix[Phoenix<br>LLM_trace<br>visualization]
        end
    end

    NIM[NVIDIA_NIM<br>Cloud_API]

    Consumer --> NGINX
    NGINX --> UI
    NGINX --> Merchant
    NGINX --> PSP
    NGINX --> AppsSDK

    UI -->|ACP_REST| Merchant
    UI -->|UCP_A2A| Merchant
    UI -->|payment_delegation| PSP
    AppsSDK -->|checkout_sessions| Merchant
    AppsSDK -->|payment_processing| PSP
    Merchant -->|process_payment| PSP
    Merchant -->|promotion_queries| Promo
    Merchant -->|post_purchase_msgs| PostPurch
    AppsSDK -->|recommendations| Reco
    AppsSDK -->|product_search| Search

    Merchant --> SQLite
    PSP --> SQLite
    Reco --> Milvus
    Search --> Milvus

    Promo --> NIM
    PostPurch --> NIM
    Reco --> NIM
    Search --> NIM

    Promo --> Phoenix
    PostPurch --> Phoenix
    Reco --> Phoenix
    Search --> Phoenix
```

### 1.3 Level 3: Component Diagram — Merchant Service

```mermaid
graph TB
    subgraph Merchant_API_Service
        subgraph Entry_Points
            ACPRouter[ACP_Checkout_Router<br>REST_endpoints_for<br>session_lifecycle]
            UCPRouter[UCP_Router<br>A2A_JSON_RPC_and<br>discovery_endpoints]
            SharedRouter[Shared_Router<br>Products_health<br>and_metrics]
        end

        subgraph Middleware_Stack
            CORSMw[CORS_Middleware]
            LogMw[Request_Logging<br>Middleware]
            ACPMw[ACP_Headers<br>Middleware<br>Idempotency_RequestId]
        end

        subgraph Domain_Layer
            CheckoutDomain[Checkout_Domain<br>Protocol_agnostic<br>business_models]
            CheckoutOps[Checkout_Operations<br>Create_update<br>complete_cancel]
        end

        subgraph Service_Layer
            PromotionSvc[Promotion_Service<br>Signal_computation<br>agent_integration]
            PostPurchSvc[Post_Purchase_Service<br>Multilingual_message<br>generation]
            IdempotencySvc[Idempotency_Service<br>Request_deduplication]
            MetricsSvc[Metrics_Services<br>Agent_outcomes_and<br>recommendation_attribution]
        end

        subgraph Data_Access
            DBModels[Database_Models<br>SQLModel_entities]
            DBSession[Session_Manager<br>SQLAlchemy_engine]
        end
    end

    CORSMw --> LogMw --> ACPMw
    ACPMw --> ACPRouter
    ACPMw --> UCPRouter
    ACPMw --> SharedRouter
    ACPRouter --> CheckoutOps
    UCPRouter --> CheckoutOps
    CheckoutOps --> CheckoutDomain
    CheckoutOps --> PromotionSvc
    CheckoutOps --> PostPurchSvc
    SharedRouter --> MetricsSvc
    ACPRouter --> IdempotencySvc
    PromotionSvc --> DBModels
    CheckoutOps --> DBModels
    DBModels --> DBSession
```

### 1.4 Level 3: Component Diagram — Apps SDK

```mermaid
graph TB
    subgraph Apps_SDK_MCP_Server
        subgraph MCP_Transport
            HTTPTransport[Stateless_HTTP<br>MCP_Transport]
        end

        subgraph MCP_Tools
            SearchTool[search_products<br>Semantic_product_search]
            RecoTool[get_recommendations<br>ARAG_personalized_recs]
            AddCartTool[add_to_cart<br>Add_item_to_cart]
            RemoveCartTool[remove_from_cart<br>Remove_cart_item]
            UpdateCartTool[update_cart_quantity<br>Change_item_qty]
            GetCartTool[get_cart<br>Retrieve_cart_state]
            CheckoutTool[checkout<br>Full_checkout_flow]
            TrackTool[track_recommendation_click<br>Attribution_event]
            CreateSessionTool[create_acp_session<br>Start_ACP_checkout]
            UpdateSessionTool[update_acp_session<br>Modify_ACP_session]
        end

        subgraph REST_Endpoints
            CartREST[Cart_REST_API<br>add_update_shipping]
            CheckoutREST[Checkout_REST<br>Process_purchase]
            SessionREST[Session_REST<br>Manage_active_sessions]
            EventsREST[Events_SSE<br>Real_time_stream]
        end

        subgraph Internal_Services
            CartManager[Cart_Manager<br>In_memory_cart_store]
            EventEmitter[Event_Emitter<br>SSE_event_bus]
            RecoHelpers[Recommendation_Helpers<br>Attribution_tracking]
        end
    end

    HTTPTransport --> MCP_Tools
    MCP_Tools --> CartManager
    MCP_Tools --> EventEmitter
    REST_Endpoints --> CartManager
    REST_Endpoints --> EventEmitter
    RecoTool --> RecoHelpers
    TrackTool --> RecoHelpers
```

### 1.5 Level 3: Component Diagram — Recommendation Agent

```mermaid
graph TB
    subgraph Recommendation_Agent_ARAG
        subgraph Entry
            NATServe[NAT_HTTP_Server<br>Port_8004]
        end

        subgraph Pipeline_Stages
            RAGRetriever[RAG_Retriever<br>Custom_component<br>Milvus_vector_search]
            ParallelExec[Parallel_Execution<br>Custom_component<br>asyncio_gather]

            subgraph Parallel_Agents
                NLIAgent[NLI_Agent<br>Natural_language<br>inference_filter]
                UUAAgent[UUA_Agent<br>User_understanding<br>intent_analysis]
            end

            ContextSummary[Context_Summary_Agent<br>Synthesize_NLI_and_UUA]
            ItemRanker[Item_Ranker<br>LLM_based_ranking<br>top_3_selection]
            OutputGuard[Output_Contract_Guard<br>Custom_component<br>Schema_validation]
        end

        subgraph External_Deps
            MilvusDB[(Milvus_Vector_DB)]
            NIMEndpoint[NVIDIA_NIM<br>LLM_and_Embeddings]
        end
    end

    NATServe --> RAGRetriever
    RAGRetriever --> MilvusDB
    RAGRetriever --> ParallelExec
    ParallelExec --> NLIAgent
    ParallelExec --> UUAAgent
    NLIAgent --> ContextSummary
    UUAAgent --> ContextSummary
    ContextSummary --> ItemRanker
    ItemRanker --> OutputGuard
    NLIAgent --> NIMEndpoint
    UUAAgent --> NIMEndpoint
    ItemRanker --> NIMEndpoint
```

---

## 2. TOGAF-Aligned Architecture Views

### 2.1 Business Architecture View

Mapping of business capabilities to system components.

```mermaid
graph TB
    subgraph Business_Capabilities
        ProductDiscovery[Product_Discovery<br>Search_browse_recommend]
        CartManagement[Cart_Management<br>Add_remove_update_items]
        CheckoutProcess[Checkout_Processing<br>Session_fulfillment_payment]
        DynamicPricing[Dynamic_Pricing<br>AI_driven_promotions]
        PaymentProcessing[Payment_Processing<br>Tokenized_card_payments]
        PostPurchaseComms[Post_Purchase_Communications<br>Order_status_messages]
        AgentOrchestration[Agent_Orchestration<br>Multi_agent_coordination]
        ProtocolCompliance[Protocol_Compliance<br>ACP_UCP_standards]
    end

    subgraph Application_Components
        UIApp[UI_Application]
        MerchantApp[Merchant_Application]
        PSPApp[PSP_Application]
        AppsSDKApp[Apps_SDK_Application]
        PromoApp[Promotion_Agent]
        RecoApp[Recommendation_Agent]
        SearchApp[Search_Agent]
        PostPurchApp[Post_Purchase_Agent]
    end

    ProductDiscovery --> SearchApp
    ProductDiscovery --> RecoApp
    ProductDiscovery --> AppsSDKApp
    CartManagement --> AppsSDKApp
    CheckoutProcess --> MerchantApp
    CheckoutProcess --> UIApp
    DynamicPricing --> PromoApp
    DynamicPricing --> MerchantApp
    PaymentProcessing --> PSPApp
    PostPurchaseComms --> PostPurchApp
    AgentOrchestration --> AppsSDKApp
    AgentOrchestration --> MerchantApp
    ProtocolCompliance --> MerchantApp
```

### 2.2 Application Architecture View

Logical grouping of applications and their interactions.

```mermaid
graph LR
    subgraph Presentation_Layer
        WebUI[Web_UI<br>Next_js_15_React_19]
    end

    subgraph Integration_Layer
        NGINX_GW[NGINX_Gateway<br>Routing_and_proxy]
        SSE_Bus[SSE_Event_Bus<br>Real_time_events]
    end

    subgraph Business_Logic_Layer
        MerchantBL[Merchant_Business_Logic<br>Checkout_domain_operations]
        PaymentBL[Payment_Business_Logic<br>Vault_and_intent_processing]
        CartBL[Cart_Business_Logic<br>In_memory_cart_operations]
    end

    subgraph AI_Services_Layer
        PromotionAI[Promotion_AI<br>Dynamic_pricing_decisions]
        RecommendationAI[Recommendation_AI<br>ARAG_personalization]
        SearchAI[Search_AI<br>Semantic_retrieval]
        PostPurchaseAI[Post_Purchase_AI<br>Message_generation]
    end

    subgraph Data_Layer
        RelationalDB[(Relational_Store<br>SQLite)]
        VectorDB[(Vector_Store<br>Milvus)]
    end

    subgraph External_Services
        LLM_API[LLM_API<br>NVIDIA_NIM]
        Observability[Observability<br>Phoenix_OTLP]
    end

    WebUI --> NGINX_GW
    NGINX_GW --> MerchantBL
    NGINX_GW --> PaymentBL
    NGINX_GW --> CartBL
    CartBL --> SSE_Bus
    SSE_Bus --> WebUI

    MerchantBL --> PromotionAI
    MerchantBL --> PostPurchaseAI
    CartBL --> RecommendationAI
    CartBL --> SearchAI
    MerchantBL --> PaymentBL

    MerchantBL --> RelationalDB
    PaymentBL --> RelationalDB
    RecommendationAI --> VectorDB
    SearchAI --> VectorDB

    PromotionAI --> LLM_API
    RecommendationAI --> LLM_API
    SearchAI --> LLM_API
    PostPurchaseAI --> LLM_API

    PromotionAI --> Observability
    RecommendationAI --> Observability
    SearchAI --> Observability
    PostPurchaseAI --> Observability
```

### 2.3 Technology Architecture View

Maps technology components to infrastructure.

```mermaid
graph TB
    subgraph Compute_Platform
        subgraph Docker_Compose_Orchestration
            subgraph App_Network
                NGINXContainer[NGINX_Container<br>Port_80]
                MerchantContainer[Merchant_Container<br>Python_3_12_FastAPI<br>Port_8000]
                PSPContainer[PSP_Container<br>Python_3_12_FastAPI<br>Port_8001]
                AppsSDKContainer[Apps_SDK_Container<br>Python_3_12_FastAPI<br>Port_2091]
                UIContainer[UI_Container<br>Node_18_Next_js<br>Port_3000]
                PromoContainer[Promo_Agent_Container<br>Python_3_12_NAT<br>Port_8002]
                PostPurchContainer[Post_Purchase_Container<br>Python_3_12_NAT<br>Port_8003]
                RecoContainer[Reco_Agent_Container<br>Python_3_12_NAT<br>Port_8004]
                SearchContainer[Search_Agent_Container<br>Python_3_12_NAT<br>Port_8005]
            end

            subgraph Infra_Network
                MilvusContainer[Milvus_Standalone<br>Port_19530]
                MinIOContainer[MinIO_Object_Store<br>Port_9000]
                EtcdContainer[Etcd_Metadata<br>Port_2379]
                PhoenixContainer[Phoenix_Observability<br>Port_6006]
            end
        end

        subgraph Persistent_Storage
            ACPVolume[acp_data_volume<br>SQLite_databases]
            MilvusVolume[milvus_data_volume<br>Vector_indexes]
            MinIOVolume[minio_data_volume<br>Object_storage]
            EtcdVolume[etcd_data_volume<br>Metadata]
        end
    end

    subgraph External_Cloud
        NVIDIA_NIM_API[NVIDIA_NIM_API<br>LLM_Inference<br>Embedding_Service]
    end

    MerchantContainer --> ACPVolume
    PSPContainer --> ACPVolume
    MilvusContainer --> MilvusVolume
    MinIOContainer --> MinIOVolume
    EtcdContainer --> EtcdVolume
    MilvusContainer --> MinIOContainer
    MilvusContainer --> EtcdContainer
    PromoContainer --> NVIDIA_NIM_API
    PostPurchContainer --> NVIDIA_NIM_API
    RecoContainer --> NVIDIA_NIM_API
    SearchContainer --> NVIDIA_NIM_API
    RecoContainer --> MilvusContainer
    SearchContainer --> MilvusContainer
```

### 2.4 Data Architecture View

```mermaid
graph TB
    subgraph Data_Domains
        subgraph Product_Domain
            ProductCatalog[Product_Catalog<br>4_products_with_pricing<br>lifecycle_demand_data]
            CompetitorPrices[Competitor_Prices<br>Price_comparison<br>from_other_retailers]
            ProductEmbeddings[Product_Embeddings<br>Vector_representations<br>for_semantic_search]
        end

        subgraph Customer_Domain
            CustomerRecords[Customer_Records<br>Email_name<br>identity]
            BrowsingHistory[Browsing_History<br>Category_views<br>search_terms]
        end

        subgraph Commerce_Domain
            CheckoutSessions[Checkout_Sessions<br>Session_lifecycle<br>items_totals_status]
            VaultTokens[Vault_Tokens<br>Tokenized_payment<br>methods]
            PaymentIntents[Payment_Intents<br>Charge_processing<br>records]
        end

        subgraph Analytics_Domain
            AgentOutcomes[Agent_Invocation_Outcomes<br>Success_rates_latency<br>error_tracking]
            AttributionEvents[Recommendation_Attribution<br>Impressions_clicks<br>purchases]
        end

        subgraph Transient_Data
            CartState[Cart_State<br>In_memory<br>session_scoped]
            SSEEvents[SSE_Event_History<br>In_memory<br>clearable]
            IdempotencyCache[Idempotency_Cache<br>In_memory<br>24h_TTL]
        end
    end

    subgraph Storage_Systems
        SQLiteDB[(SQLite<br>Relational)]
        MilvusDB[(Milvus<br>Vector)]
        Memory[(Process_Memory<br>Ephemeral)]
    end

    ProductCatalog --> SQLiteDB
    CompetitorPrices --> SQLiteDB
    CustomerRecords --> SQLiteDB
    BrowsingHistory --> SQLiteDB
    CheckoutSessions --> SQLiteDB
    VaultTokens --> SQLiteDB
    PaymentIntents --> SQLiteDB
    AgentOutcomes --> SQLiteDB
    AttributionEvents --> SQLiteDB
    ProductEmbeddings --> MilvusDB
    CartState --> Memory
    SSEEvents --> Memory
    IdempotencyCache --> Memory
```

---

## 3. Integration Diagrams

### 3.1 End-to-End Checkout Sequence (ACP Protocol)

```mermaid
sequenceDiagram
    participant User as Consumer
    participant UI as Next_js_UI
    participant NGINX as NGINX_Gateway
    participant Merchant as Merchant_API
    participant Promo as Promotion_Agent
    participant NIM as NVIDIA_NIM
    participant PSP as PSP_Payment
    participant PostPurch as Post_Purchase_Agent
    participant SSE as SSE_Event_Bus

    User->>UI: select_product_and_add_to_cart
    UI->>NGINX: POST_checkout_sessions
    NGINX->>Merchant: forward_request

    loop For each line item
        Merchant->>Merchant: compute_business_signals
        Merchant->>Promo: request_promotion_decision
        Promo->>NIM: LLM_inference
        NIM-->>Promo: action_and_reasoning
        Promo-->>Merchant: promotion_decision
        Merchant->>Merchant: validate_and_apply_discount
    end

    Merchant-->>NGINX: session_with_promotions
    NGINX-->>UI: display_session
    UI->>SSE: emit_checkout_and_agent_events
    SSE-->>UI: real_time_protocol_log

    User->>UI: enter_address_select_shipping
    UI->>NGINX: POST_checkout_sessions_id_update
    NGINX->>Merchant: update_fulfillment
    Merchant-->>NGINX: status_ready_for_payment
    NGINX-->>UI: updated_session

    User->>UI: enter_payment_details
    UI->>NGINX: POST_delegate_payment
    NGINX->>PSP: create_vault_token
    PSP-->>NGINX: vault_token
    NGINX-->>UI: token_received

    UI->>NGINX: POST_checkout_sessions_id_complete
    NGINX->>Merchant: complete_with_token
    Merchant->>PSP: process_payment_intent
    PSP-->>Merchant: payment_confirmed

    Merchant->>PostPurch: generate_order_message
    PostPurch->>NIM: LLM_multilingual_generation
    NIM-->>PostPurch: localized_message
    PostPurch-->>Merchant: shipping_message

    Merchant->>UI: POST_webhook_order_completed
    Merchant-->>NGINX: completed_session_with_order
    NGINX-->>UI: order_confirmation

    UI->>User: display_order_confirmation
```

### 3.2 Recommendation Flow Sequence

```mermaid
sequenceDiagram
    participant UI as Next_js_UI
    participant SDK as Apps_SDK_MCP
    participant Reco as Recommendation_Agent
    participant Milvus as Milvus_Vector_DB
    participant NIM as NVIDIA_NIM
    participant Merchant as Merchant_API

    UI->>SDK: MCP_get_recommendations
    Note right of UI: product_id_product_name_cart_items

    SDK->>SDK: emit_recommendation_pending_event
    SDK->>Reco: HTTP_POST_recommendation_request

    Reco->>Milvus: vector_similarity_search
    Milvus-->>Reco: candidate_products

    par Parallel Execution
        Reco->>NIM: NLI_relevance_filtering
        NIM-->>Reco: filtered_candidates
    and
        Reco->>NIM: UUA_user_intent_analysis
        NIM-->>Reco: user_intent_signals
    end

    Reco->>NIM: context_synthesis
    NIM-->>Reco: synthesized_context

    Reco->>NIM: item_ranking_top_3
    NIM-->>Reco: ranked_recommendations

    Reco->>Reco: output_contract_guard_validation
    Reco-->>SDK: 3_recommendations_with_reasoning

    SDK->>SDK: emit_recommendation_complete_event
    SDK->>Merchant: record_agent_outcome_metric
    SDK-->>UI: recommendations_with_pipeline_trace

    UI->>UI: display_recommendation_cards
    User->>UI: click_recommended_product
    UI->>SDK: track_recommendation_click
    SDK->>Merchant: record_attribution_click_event
```

### 3.3 UCP Discovery and A2A Flow

```mermaid
sequenceDiagram
    participant Agent as Client_Agent
    participant Merchant as Merchant_API

    Agent->>Merchant: GET_well_known_ucp
    Merchant-->>Agent: UCPBusinessProfile
    Note left of Merchant: capabilities_services_payment_handlers_signing_keys

    Agent->>Agent: negotiate_capabilities
    Note right of Agent: compute_intersection_of<br>agent_and_merchant_capabilities

    Agent->>Merchant: POST_a2a_JSON_RPC
    Note right of Agent: method_a2a_ucp_checkout_create
    Merchant->>Merchant: create_checkout_session_protocol_ucp
    Merchant-->>Agent: JSON_RPC_result_with_contextId

    Agent->>Merchant: POST_a2a_JSON_RPC
    Note right of Agent: method_a2a_ucp_checkout_update
    Merchant-->>Agent: updated_checkout_state

    Agent->>Merchant: POST_a2a_JSON_RPC
    Note right of Agent: method_a2a_ucp_checkout_complete
    Merchant-->>Agent: completed_order_with_UCP_metadata
```

---

## 4. Logical Architecture Diagram (TOGAF)

### 4.1 Logical Component Model

```mermaid
graph TB
    subgraph Logical_Architecture
        subgraph Interaction_Layer
            AgentInterface[Agent_Interaction_Interface<br>Protocol_agnostic_agent<br>communication_layer]
            UserInterface[User_Interaction_Interface<br>Visual_dashboard_for<br>monitoring_and_testing]
        end

        subgraph Protocol_Layer
            ACPAdapter[ACP_Protocol_Adapter<br>REST_based_checkout<br>session_management]
            UCPAdapter[UCP_Protocol_Adapter<br>A2A_JSON_RPC_with<br>discovery_and_negotiation]
            MCPAdapter[MCP_Protocol_Adapter<br>Tool_based_agent<br>interaction_standard]
        end

        subgraph Business_Services_Layer
            CheckoutService[Checkout_Service<br>Session_lifecycle<br>management]
            PricingService[Pricing_Service<br>Dynamic_promotion<br>engine]
            FulfillmentService[Fulfillment_Service<br>Shipping_options<br>and_selection]
            PaymentService[Payment_Service<br>Tokenized_payment<br>processing]
            CatalogService[Catalog_Service<br>Product_data<br>and_inventory]
            CommunicationService[Communication_Service<br>Post_purchase<br>messaging]
        end

        subgraph Intelligence_Layer
            PromotionIntelligence[Promotion_Intelligence<br>Signal_based_LLM<br>arbitration]
            RecommendationIntelligence[Recommendation_Intelligence<br>ARAG_personalization<br>pipeline]
            SearchIntelligence[Search_Intelligence<br>Semantic_vector<br>retrieval]
            MessageIntelligence[Message_Intelligence<br>Multilingual<br>generation]
        end

        subgraph Data_Services_Layer
            RelationalDataService[Relational_Data_Service<br>CRUD_operations_for<br>structured_entities]
            VectorDataService[Vector_Data_Service<br>Embedding_storage<br>and_similarity_search]
            EventDataService[Event_Data_Service<br>Real_time_event<br>streaming]
            MetricsDataService[Metrics_Data_Service<br>Agent_performance<br>and_attribution]
        end
    end

    AgentInterface --> ACPAdapter
    AgentInterface --> UCPAdapter
    AgentInterface --> MCPAdapter
    UserInterface --> ACPAdapter
    UserInterface --> UCPAdapter

    ACPAdapter --> CheckoutService
    UCPAdapter --> CheckoutService
    MCPAdapter --> CatalogService

    CheckoutService --> PricingService
    CheckoutService --> FulfillmentService
    CheckoutService --> PaymentService
    CheckoutService --> CommunicationService

    PricingService --> PromotionIntelligence
    PricingService --> CatalogService
    CatalogService --> SearchIntelligence
    CatalogService --> RecommendationIntelligence
    CommunicationService --> MessageIntelligence

    CheckoutService --> RelationalDataService
    PaymentService --> RelationalDataService
    RecommendationIntelligence --> VectorDataService
    SearchIntelligence --> VectorDataService
    CheckoutService --> EventDataService
    PricingService --> MetricsDataService
    RecommendationIntelligence --> MetricsDataService
```

### 4.2 Cross-Cutting Concerns

```mermaid
graph LR
    subgraph Cross_Cutting
        Security[Security<br>API_key_auth<br>CORS_vault_tokens]
        Idempotency[Idempotency<br>Hash_based<br>deduplication]
        Observability[Observability<br>Phoenix_OTLP<br>request_logging]
        ErrorHandling[Error_Handling<br>Fail_closed<br>graceful_fallbacks]
        Configuration[Configuration<br>Environment_based<br>Docker_secrets]
    end

    subgraph All_Services
        Merchant[Merchant]
        PSP[PSP]
        AppsSDK[Apps_SDK]
        Agents[AI_Agents]
    end

    Security --> Merchant
    Security --> PSP
    Security --> AppsSDK
    Idempotency --> Merchant
    Idempotency --> PSP
    Observability --> Agents
    Observability --> Merchant
    ErrorHandling --> Merchant
    ErrorHandling --> AppsSDK
    ErrorHandling --> Agents
    Configuration --> Merchant
    Configuration --> PSP
    Configuration --> AppsSDK
    Configuration --> Agents
```
