# API Contracts Specification

## 1. Public API (Frontend Web)

### 1.1. Create Quote (Multi-Currency)
  **Endpoint:** `POST /api/public/quotes`

  **Description:** Receives a shopping cart from the public web and generates a quote using the daily exchange rate for the requested currency. Accepts customer city and address for physical invoice and dispatch matching.

  **Request Body (JSON):**

    ```json
    {
      "customer": {
        "tax_id": "J-12345678-9",
        "business_name": "Inversiones El Sol C.A.",
        "email": "contacto@elsol.com"
      },
      "customer_city": "Caracas",
      "customer_address": "Av. Francisco de Miranda, Torre Delta, Piso 4",
      "target_currency": "VES", 
      "items": [
        { "sku": "EXT-PQS-10", "quantity": 2, "type": "PRODUCT" },
        { "sku": "REC-CO2-5", "quantity": 1, "type": "SERVICE" }
      ]
    }
    ```
  **Response Body (JSON):**

    ```json
    {
      "quote_id": "a1b2c3d4-...",
      "status": "PENDING",
      "target_currency": "VES",
      "exchange_rate": 36.50,
      "total_amount_usd": 100.00,
      "total_amount_local": 3650.00,
      "customer_city": "Caracas",
      "customer_address": "Av. Francisco de Miranda, Torre Delta, Piso 4",
      "document_number": null,
      "message": "Quote generated successfully."
    }
    ```

### 1.2. Get Catalog
  **Endpoint:** `GET /api/public/catalog`

  **Description:** Returns the list of products and services. Prices are always returned in the base currency (USD). The React frontend is responsible for displaying the estimated local price.


---

## 2. Admin API (Private Panel)

### 2.1. Issue Paid Invoice from Quote
*   **Endpoint:** `POST /api/admin/invoices/pay-and-issue/{quote_id}`
*   **Description:** Issues a paid and frozen invoice using the exchange rate from the day of payment.
*   **Request Body (JSON):**
    ```json
    {
      "target_currency": "VES",
      "exchange_rate": 45.50,
      "payment_reference": "0102-123456789"
    }
    ```
*   **Response Body (JSON):**
    ```json
    {
      "invoice_id": "f5a6b7c8-...",
      "quote_id": "a1b2c3d4-...",
      "status": "PAID",
      "target_currency": "VES",
      "exchange_rate": 45.50,
      "total_amount_usd": 100.00,
      "total_amount_local": 4550.00,
      "rate_date": "2026-10-16T10:00:00Z",
      "payment_reference": "0102-123456789"
    }
    ```

### 2.2. Get Invoices (List & Details)
*   **List Invoices Endpoint:** `GET /api/admin/invoices`
*   **Query Parameters:** `page` (default 0), `size` (default 10)
*   **Description:** Returns a paginated list of all invoices in the system with their status, currency, and local converted amounts.
*   **Single Invoice Endpoint:** `GET /api/admin/invoices/{id}`
*   **Description:** Returns the details of a specific invoice by its UUID.

### 2.3. Get Quotes (List & Details)
*   **List Quotes Endpoint:** `GET /api/admin/quotes`
*   **Query Parameters:** `page` (default 0), `size` (default 10), `status` (optional: PENDING, APPROVED, INVOICED)
*   **Description:** Returns a paginated list of quotes, allowing administrators to pick an approved quote ID to test invoicing.
*   **Single Quote Endpoint:** `GET /api/public/quotes/{id}`
*   **Description:** Returns the details of a specific quote.

### 2.4. MCP Chat (AI Assistant)
  **Endpoint:** `POST /api/admin/chat` (Legacy) / `POST /api/admin/chat/mcp` (Tool-Calling)

  **Headers:** `Authorization: Bearer <JWT_TOKEN>`

  **Description:** Sends a natural language prompt to the Spring Boot MCP Server. The backend orchestrates pgvector/SQL tools and returns the LLM response.

  **Request Body (JSON):**

    ```json
    {
      "prompt": "¿Cuáles son los clientes que tienen extintores vencidos este mes?"
    }
    ```
  **Response Body (JSON):**

    ```json
    {
      "reply": "Hay 3 clientes con extintores vencidos: Inversiones El Sol (J-123...), ...",
      "tools_used": ["getCustomerHistory"],
      "sources": [] 
    }
    ```

### 2.5. Upload Document for RAG Vectorization
  **Endpoint:** `POST /api/admin/documents/upload`

  **Headers:** `Content-Type: multipart/form-data`

  **Description:** Uploads a PDF manual, standard, or technical guideline. Automatically purges prior embeddings matching the specified category and domain (`DOCUMENT`), splits content into semantic chunks, and persists embeddings into `pgvector`.

  **Form Parameters:**
  * `file`: MultipartFile (binary `.pdf` file)
  * `category`: String (e.g. `NFPA10`, `COVENIN`, `SAFETY`, `TURISMO`)

  **Response Body (JSON):**
  ```json
  {
    "file_name": "manual_nfpa_10.pdf",
    "category": "NFPA10",
    "chunks_count": 18,
    "status": "SUCCESS",
    "message": "Documento manual_nfpa_10.pdf procesado y vectorizado exitosamente en la categoría NFPA10 (18 fragmentos)."
  }
  ```

### 2.6. Authentication & Session Control

#### User Login
*   **Endpoint:** `POST /api/auth/login`
*   **Access:** Public
*   **Description:** Authenticates user credentials. Issues a signed JWT stored strictly within an `HttpOnly` cookie (`jwt_token`). The token is not returned in the JSON response body to protect against client-side script extraction (XSS).
*   **Request Body (JSON):**
    ```json
    {
      "username": "admin",
      "password": "adminpassword"
    }
    ```
*   **Response Headers:**
    ```http
    Set-Cookie: jwt_token=eyJhbGciOiJIUzI1Ni...; Path=/; HttpOnly; SameSite=Lax
    ```
*   **Response Body (JSON):**
    ```json
    {
      "message": "Login successful",
      "username": "admin",
      "role": "ADMIN"
    }
    ```

#### User Logout
*   **Endpoint:** `POST /api/auth/logout`
*   **Access:** Public
*   **Description:** Clears the authentication `jwt_token` cookie by setting `Max-Age=0`.
*   **Response Body (JSON):**
    ```json
    {
      "message": "Logged out successfully"
    }
    ```
