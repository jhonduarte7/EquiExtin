# System Specification: Fire Safety Management, Invoicing, and AI Assistant (MCP)

## 1. General Architecture
*   **Primary Database:** PostgreSQL 15+ with the `pgvector` extension enabled.
*   **Backend:** Spring Boot (Java 25). Acts as the REST API Server and **MCP (Model Context Protocol) Server**.
*   **Public Frontend:** React. E-commerce style catalog and quote request cart (no payment gateway).
*   **Admin Frontend:** React. Private control panel + **MCP Client** (AI Chatbot).
*   **Deployment Target:** AWS (Backend/Containers) and Supabase (Managed Database).

### 1.1. Backend Package Architecture (Modular Monolith & Clean Architecture)
To ensure isolation, high cohesion, strict data mapping, and prepare for future domain extractions, the backend follows a Modular Monolith layout under `com.extin.backend`:

-   `config/`: Spring Boot configurations, Security, Async, and Data initializers (`DataInitializer`, `DatabaseSeeder`).
-   `controllers/`: REST Controllers exposing public and administrative endpoints.
-   `exceptions/`: Global error handling and custom domain exceptions.
-   `mappers/`: Dedicated classes or interfaces responsible for transforming Entities to DTOs and vice versa, ensuring the API contracts remain decoupled from the database schema.
-   `models/`: Domain Objects strictly separated by their responsibility:
    *   `dtos/`: Immutable Data Transfer Objects representing API contracts, implemented strictly as Java `record`.
    *   `entities/`: Core JPA domain models (`@Entity`), value objects, and lifecycle listeners.
    *   `enums/`: Shared domain enumerations.
-   `repositories/`: Spring Data JPA interfaces for transactional persistence.
-   `services/`: Core transactional and domain business logic.
-   `mcp/`: Isolated MCP module:
    *   `mcp/client/`:
        *   `config/`: Infrastructure configuration for AI clients (`McpClientConfig` declaring `ChatClient` and registering tools).
    *   `mcp/server/`:        
        *   `tools/`: Domain-specific tools grouped by entity (`CatalogTools`, `CustomerTools`).

**Separation of AI Concerns (Clean Architecture):**
-   **Infrastructure Configuration (`mcp/client/config/McpClientConfig`):** Exposes the `ChatClient` bean configured with its builder and default tools. Contains NO domain prompts or business logic.
-   **Domain Business Logic & System Prompt (`services/ChatService`):** Encapsulates the official `SYSTEM_PROMPT` containing fire safety behavioral rules, formatting guidelines, and prompt execution.
-   **HTTP Presentation Layer (`controllers/McpChatController`):** REST endpoint (`POST /api/admin/chat/mcp`) that receives JSON payloads, delegates directly to `ChatService`, and returns responses in Markdown or JSON.

## 2. Core Relational Model (Transactional Tables)

### 2.1. Customer and Equipment Management
*   **`customers`**
    *   `id` (UUID, PK)
    *   `tax_id` (String, Unique) -> Venezuelan RIF or Cédula (e.g., J-12345678-9, V-12345678)
    *   `business_name` (String)
    *   `address`, `phone`, `email` (Strings)
*   **`customer_equipment`** (Tracks client's physical extinguishers for maintenance/refills)
    *   `id` (UUID, PK)
    *   `customer_id` (FK -> customers)
    *   `equipment_type` (String) -> e.g., "PQS 10 Lbs", "CO2 5 Lbs"
    *   `last_refill_date` (Date)
    *   `expiration_date` (Date)

### 2.2. Catalog (Products and Services)
*   **`catalog_items`**
    *   `id` (UUID, PK)
    *   `sku_code` (String, Unique)
    *   `name` (String)
    *   `type` (Enum: PRODUCT, SERVICE) -> PRODUCT = New equipment sales; SERVICE = Refills/Maintenance.
    *   `base_price` (Decimal)
    *   `image_url` (String) -> Cloud storage path for the public web gallery.

### 2.3. Commercial Workflow (Quote to Invoice)
-   **`quotes`**
    *   `id` (UUID, PK)
    *   `customer_id` (FK -> customers)
    *   `customer_city` (String, Length 100): Customer city for physical document dispatch.
    *   `customer_address` (String, Length 500): Customer physical delivery / billing address.
    *   `document_number` (String, Length 50): Correlative order / invoice paper book number (número correlativo de talonario físico, ej: "003877").
    *   `issue_date` (Timestamp)
    *   `status` (Enum: PENDING, APPROVED, REJECTED, INVOICED) -> INVOICED: Converted into an official cash invoice (factura de contado).
    *   `base_currency` (String, Length 3, default USD)
    *   `target_currency` (String, Length 3)
    *   `exchange_rate` (Decimal)
    *   `total_amount` (Decimal)
    *   `converted_total_amount` (Decimal)
-   **`quote_details`**
    *   `id` (UUID, PK), `quote_id` (FK), `item_id` (FK), `quantity` (Int), `unit_price` (Decimal).
-   **`invoices`** (Generated immutably upon payment notification)
    *   `id` (UUID, PK)
    *   `quote_id` (UUID, FK -> quotes)
    *   `status` (String): PAID
    *   `target_currency` (String): USD, EUR, VES
    *   `exchange_rate` (Decimal): Applied rate at the time of payment.
    *   `total_amount_usd` (Decimal): Immutable base amount.
    *   `total_amount_local` (Decimal): Billed amount (`total_amount_usd` * `exchange_rate`).
    *   `rate_date` (Timestamp): Exact date and time the payment was reported.
    *   `payment_reference` (String): Bank transfer or mobile payment reference.
-   **`invoice_details`**
    *   Matches the structure of `quote_details`.

### 2.4. Automated JPA Auditing (`Auditory` Superclass)
To guarantee auditability across transactional mutations, entities inherit from the `Auditory` mapped superclass:
*   **Class Annotations:** `@MappedSuperclass`, `@EntityListeners(AuditingEntityListener.class)`.
*   **Audit Fields:**
    *   `created_by` (`String`, `@CreatedBy`, `updatable = false`): Creator's email or username.
    *   `last_modified_by` (`String`, `@LastModifiedBy`): Last modifier's email or username.
    *   `created_at` (`LocalDateTime`, `@CreatedDate`, `updatable = false`): Record creation timestamp.
    *   `updated_at` (`LocalDateTime`, `@LastModifiedDate`): Record modification timestamp.
*   **Auditor Resolution (`AuditorAwareImpl`):**
    *   Extracts current `Authentication` from `SecurityContextHolder`.
    *   Authenticated users: Returns `Optional.of(user.getEmail())` (or `user.getUsername()`).
    *   Internal / Unauthenticated operations (e.g., `DataInitializer`, `DatabaseSeeder`): Returns `Optional.of("SYSTEM")`.
*   **Audited Entities:** `Customer`, `CatalogItem`, `Invoice`, `Quote`, `User`, `CustomerEquipment`, `InvoiceDetail`, `QuoteDetail`.

---

## 3. Vector Model and Semantic Search (pgvector)

To enable the AI assistant in the administrative panel, the following tables combine transactional data with vector embeddings:

### 3.1. Catalog Embeddings (For fuzzy search and recommendations)
*   **`catalog_embeddings`**
    *   `item_id` (FK -> catalog_items)
    *   `enriched_description` (Text) -> e.g., "10 lb Dry Chemical (PQS) Fire Extinguisher, ideal for ABC class fires (wood, paper, electrical)."
    *   `vector_embedding` (Vector: 384 dimensions for all-minilm)

### 3.2. RAG Document Embeddings (Manuals, Safety Regulations)
*   **`rag_documents`** (PDF files uploaded via the Admin Panel)
    *   `id` (UUID, PK), `file_name`, `upload_date`
*   **`document_chunks`**
    *   `id` (UUID), `document_id` (FK)
    *   `text_content` (Text) -> Extracted text fragment.
    *   `vector_embedding` (Vector)

---

## 4. MCP (Model Context Protocol) Tools

The Spring Boot Backend will expose the following `Tools` to the Administrator's AI Chatbot:

1.  **`tool_semantic_catalog_search`**
    *   *Trigger:* Fuzzy or descriptive catalog inquiries ("Which extinguisher is best for a server room?").
    *   *Action:* Executes a cosine similarity search in `pgvector` against `catalog_embeddings`.
2.  **`tool_sql_customer_history`**
    *   *Trigger:* Exact metrics or historical tracking ("When does the refill expire for customer RIF J-123456?").
    *   *Action:* Executes standard SQL JOINs across `customers` and `customer_equipment`.
3.  **`tool_rag_manuals_search`**
    *   *Trigger:* Compliance or manual-based questions ("What is the standard mounting height for an extinguisher according to the uploaded manual?").
    *   *Action:* Executes a similarity search against `document_chunks`.

---

## 5. Frontend Architecture (React) and UX/UI

### 5.1. Internationalization (i18n)
*   **Library:** `react-i18next`.
*   **Supported Languages:** English (en), Spanish (es), Portuguese (pt).
*   **Strict Data Rule:** UI elements (buttons, menus, placeholders) are translated. Database-driven content (customer names, legal product descriptions) must render in its original language to maintain fiscal integrity.

### 5.2. Responsive Design & Theming
*   **CSS Framework:** Tailwind CSS (Mobile-First approach).
*   **Theming:** Light/Dark Mode implemented via Tailwind classes (`dark:bg-gray-800`), toggled via the global header and persisted in `localStorage`.

---

## 6. Security Architecture (Spring Security & HttpOnly JWT)

### 6.1. Authentication & Authorization Model
*   **Mechanism:** Strict Stateless JSON Web Tokens (JWT) using HMAC-SHA256 signature.
*   **Roles (`Role` Enum):**
    *   `ADMIN`: Full administrative access (quotes, invoices, customer management, document uploads, and MCP chat).
    *   `USER`: Standard operational access (chat queries, status checks).
*   **Domain Entity:** `User` implements Spring Security's `UserDetails` interface, mapping authorities as `ROLE_ADMIN` and `ROLE_USER`.
*   **Token Transport & Storage (HttpOnly Cookie):**
    *   Cookie Name: `jwt_token`.
    *   Flags: `HttpOnly = true` (blocks client-side JavaScript access, mitigating XSS token theft), `Secure = true` (in production / HTTPS environments), `Path = /`.
    *   **Body Privacy:** The generated JWT is **never** exposed in the response body. Only the user profile (`username`, `role`, `message`) is returned as JSON.
*   **Filter Pipeline:** Custom `JwtAuthenticationFilter` intercepts requests before `UsernamePasswordAuthenticationFilter`, reads the `jwt_token` cookie, validates the claims, and establishes the `UsernamePasswordAuthenticationToken` in the `SecurityContext`.

### 6.2. Endpoint Protection
*   **Public Routes (No Authentication Required):**
    *   `POST /api/auth/login`
    *   `POST /api/auth/logout`
    *   `GET /api/public/catalog`
    *   `POST /api/public/quotes`
    *   `GET /api/public/quotes/{id}`
*   **Admin Routes (Requires `ADMIN` Role):**
    *   `GET, POST, PUT, DELETE /api/admin/**`
    *   `POST /api/admin/chat` (MCP Chat Assistant)
    *   `POST /api/admin/chat/mcp` (Conversational MCP Chat Assistant)
    *   `POST /api/admin/documents/upload` (PDF Document RAG)
    *   `POST /api/admin/invoices/pay-and-issue/{quote_id}`
*   **Chat Routes (Allows `ADMIN` and `USER` Roles):**
    *   `/api/chat/**`

### 6.3. Session & Vulnerability Mitigation
*   **Session Management:** Strictly `SessionCreationPolicy.STATELESS` — zero HTTP session persistence on the server.
*   **CORS Configuration:** Configured with `allowCredentials(true)` and explicit allowed origins to ensure browser security models correctly transmit HttpOnly cookies cross-origin.
*   **CSRF:** Disabled (`csrf.disable()`) as the architecture is strictly stateless REST.
*   **XSS Mitigation:** Complete elimination of token extraction via `localStorage` or `sessionStorage` through HttpOnly cookie enforcement.

---

## 7. Phase 3: Artificial Intelligence, RAG & MCP Server (Modular Monolith)

### 7.1. Technology Stack & Integration
*   **Framework:** Spring AI (`spring-ai-bom`).
*   **Local LLM & Embedding Engine:** Ollama (`spring-ai-starter-model-ollama`).
*   **Vector Database:** PostgreSQL with `pgvector` extension via Spring AI's `VectorStore` (`spring-ai-starter-vector-store-pgvector`).
*   **Embedding Dimension:** 384 dimensions (default for `all-minilm` / `nomic-embed-text`).

### 7.2. Module 1: Document RAG (PDF Manuals and Safety Standards)
1.  **PDF Document Upload (`POST /api/admin/documents/upload`):**
    *   Administrative endpoint (`DocumentController`) accepting `multipart/form-data` with a PDF file (`file`) and a logical category identifier (`category`, e.g. `NFPA10`, `COVENIN`, `SAFETY`).
2.  **Category Purging & Clean Replacement (`DocumentRagService`):**
    *   **PASO 1 (Purgado):** Before indexing new vectors, executes `DELETE FROM vector_store WHERE metadata->>'domain' = 'DOCUMENT' AND metadata->>'category' = ?` to remove obsolete chunks for that specific category.
3.  **Document Ingestion, Chunking & Metadata Enrichment (`DocumentRagService`):**
    *   **PASO 2 (Extracción):** Saves the uploaded PDF temporarily and parses page contents via Spring AI's `PagePdfDocumentReader`.
    *   Splits extracted text into semantic chunks using `TokenTextSplitter`.
    *   **PASO 3 (Inyección de Metadatos):** Injects mandatory metadata into each chunk: `domain = 'DOCUMENT'`, `category = <category>`, `file_name`, `document_type = 'MANUAL_PDF'`, `chunk_index`, `total_chunks`, `upload_timestamp`.
    *   **PASO 4 (Guardado):** Persists embeddings into `VectorStore` (`vector_store` table in PostgreSQL with pgvector).
4.  **Domain-Isolated Document RAG MCP Tool (`DocumentTools.searchInManuals`):**
    *   Exposes `@Tool searchInManuals(String query)` to the AI Assistant.
    *   Executes similarity search strictly filtered by `domain == 'DOCUMENT'` to prevent collisions with catalog product embeddings.
    *   Provides citations with source file names, category, and page references.

### 7.3. Real-Time Auto-Vectorization Workflow
1.  **Entity Lifecycle Triggers:** Whenever a `CatalogItem` (or document) is persisted (`@PostPersist`) or updated (`@PostUpdate`), a JPA `@EntityListeners` component intercepts the change.
2.  **Spring Application Event:** An asynchronous `EntityVectorizationEvent` is published carrying the entity ID, type, text content, and metadata.
3.  **Vectorization Service (`VectorizationService`):**
    *   Listens to `EntityVectorizationEvent` via `@Async` and `@EventListener` (non-blocking).
    *   Formats descriptive metadata (`sku`, `name`, `type`, `base_price`, description).
    *   Generates a vector embedding through Spring AI's `EmbeddingModel` and saves it immutably into `VectorStore` (PostgreSQL `vector_store` table).
4.  **Initial Seeding (`DataInitializer`):**
    *   A `CommandLineRunner` component seeds baseline catalog items (e.g. PQS and CO2 fire extinguishers) and automatically invokes `VectorizationService` to populate initial vector representations.

### 7.4. MCP Chat Architecture, Prompt Separation & Conversational Memory (Separation of AI Concerns)
*   **Infrastructure Configuration (`McpClientConfig`):** Placed in `com.extin.backend.mcp.client.config`, responsible for declaring technical infrastructure beans such as the `ChatMemory` bean (`MessageWindowChatMemory` backed by `InMemoryChatMemoryRepository`).
*   **Business Logic, System Prompt & Conversational Memory (`ChatService`):** Placed in `com.extin.backend.services`. Encapsulates:
    - Immutable `SYSTEM_PROMPT` with strict domain boundaries (rejects out-of-scope inquiries like politics, sports, recipes, or general tourism; mandates fire safety, extinguisher catalogs, and NFPA/COVENIN standard focus; and mandates domain tools usage and Markdown formatting).
    - Injects `ChatClient.Builder`, configures `MessageChatMemoryAdvisor` bound to `ChatMemory`, and registers domain tools (`CatalogTools`, `CustomerTools`, `DocumentTools`).
    - Main chat method `processChat(String chatId, String userMessage)` isolating conversations per session via `ChatMemory.CONVERSATION_ID` parameter, maintaining contextual history while preventing cross-tenant bleed.
*   **Presentation Controller (`McpChatController`):** Placed in `com.extin.backend.controllers`, strictly handles HTTP requests on `POST /api/admin/chat/mcp`, extracts `chatId` from DTO body or `X-Chat-Id` header, and delegates execution exclusively to `ChatService`.



