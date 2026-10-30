# Equi-Extin | B2B ERP & Fire Safety Management System

[![Java](https://img.shields.io/badge/Java-21-orange.svg?logo=openjdk&logoColor=white)](https://openjdk.org/projects/jdk/21/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x%20%2F%204.x-brightgreen.svg?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Spring AI](https://img.shields.io/badge/Spring%20AI-2.0.x-blue.svg?logo=spring&logoColor=white)](https://spring.io/projects/spring-ai)
[![React](https://img.shields.io/badge/React-19.x-61DAFB.svg?logo=react&logoColor=black)](https://react.dev/)
[![Vite](https://img.shields.io/badge/Vite-8.x-646CFF.svg?logo=vite&logoColor=white)](https://vitejs.dev/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-3.4.x-38B2AC.svg?logo=tailwind-css&logoColor=white)](https://tailwindcss.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16%20%2B%20pgvector-336791.svg?logo=postgresql&logoColor=white)](https://github.com/pgvector/pgvector)
[![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub_Actions-2088FF.svg?logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![Vercel](https://img.shields.io/badge/Deploy-Vercel-black.svg?logo=vercel&logoColor=white)](https://venefast.com)
[![AWS](https://img.shields.io/badge/Cloud-AWS-232F3E.svg?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![License](https://img.shields.io/badge/License-Proprietary-red.svg)]()

Enterprise Resource Planning (ERP) platform and AI-powered industrial assistant engineered for **Equi-Extin**, a leading fire protection equipment manufacturer, maintenance workshop, and distributor. 

The platform integrates wholesale B2B commercial workflows, physical invoice slip replication (SENIAT / Contado standards), multi-currency valuation (USD and Venezuelan Official BCV Bolívares), client cylinder fleet tracking with hydro-static test (PH) expiration alerts, and an autonomous AI Technical Copilot powered by **Spring AI**, **pgvector**, and **Llama 3.1** via **Model Context Protocol (MCP)** tool calling.

---

## 1. System Architecture & Tech Stack

The system follows a modern decoupled monolith architecture designed for high availability, security, and developer ergonomics:

```
+-------------------------------------------------------------------------------------------------+
|                                        FRONTEND LAYER                                           |
|                            React 19 + Vite + Tailwind CSS + i18n                                |
|                 Hosted on Vercel Edge Network: https://venefast.com                             |
+-------------------------------------------------------------------------------------------------+
                                      |                     ^
                  REST API (JSON)     |                     |  SSE Token Stream
         withCredentials: true        v                     |  (text/event-stream)
+-------------------------------------------------------------------------------------------------+
|                                        BACKEND LAYER                                            |
|                    Spring Boot / Java 21 LTS / Stateless Spring Security                        |
|                                 Deployed on AWS Cloud                                           |
+-------------------------------------------------------------------------------------------------+
|                                                                                                 |
|   +--------------------+     +-----------------------+     +--------------------------------+   |
|   |  Controllers Layer |     |  Spring AI & MCP Tools|     |  Document RAG Engine           |   |
|   |  REST / Auth / ERP | --> |  Catalog, Customer &  | --> |  PDF Ingestion, TokenSplitter  |   |
|   |  Stateless Cookie  |     |  Document Callbacks   |     |  Domain Category Isolation     |   |
|   +--------------------+     +-----------------------+     +--------------------------------+   |
|                                         |                                  |                    |
+-----------------------------------------|----------------------------------|--------------------+
                                          v                                  v
+-------------------------------------------------------------------------------------------------+
|                                   DATA & INFERENCE LAYER                                        |
|                                                                                                 |
|    +------------------------------------------+    +---------------------------------------+    |
|    |      Local Ollama LLM Inference Engine   |    |    PostgreSQL 16 Relational & Vector  |    |
|    |  • Llama 3.1 (8B) - Natural Reasoning    |    |  • Relational: Quotes, Invoices, CRM  |    |
|    |  • all-minilm (384d) - Dense Embeddings  |    |  • pgvector: HNSW Index / Cosine      |    |
|    +------------------------------------------+    +---------------------------------------+    |
+-------------------------------------------------------------------------------------------------+
```

### Core Technologies

* **Frontend Application (`frontend-extin`):**
  - **Framework & Tooling:** React 19 (`react`, `react-dom`), Vite 8.3 with ultra-fast Hot Module Replacement (HMR).
  - **Design System:** Tailwind CSS 3.4 with full Dark / Light mode tokens, `tailwindcss-animate`, Lucide Icons, and modular UI primitives.
  - **State & Routing:** React Router DOM v7, React Context API (`AuthContext`, `CartContext`, `ThemeContext`).
  - **Localization (i18n):** `i18next` with trilingual parity: Spanish (🇪🇸 `es`), English (🇺🇸 `en`), and Portuguese (🇧🇷 `pt`).
  - **Networking:** Axios with `withCredentials: true` and native `fetch` + `ReadableStream` for real-time Server-Sent Events (SSE).

* **Backend Services (`backend-extin`):**
  - **Framework & Runtime:** Java 21 (LTS) / Java 25 binary compatible, Spring Boot 3.x / 4.x.
  - **Security:** Spring Security with 100% stateless session management and HttpOnly JWT cookie delivery (`jwt_token`).
  - **Persistence & Auditing:** Spring Data JPA with Hibernate, automated entity auditing (`Auditory` mapped superclass).
  - **Build & Quality:** Apache Maven Wrapper (`./mvnw`), 90 unit and integration tests passing with 100% success rate.

* **Artificial Intelligence & RAG Subsystem:**
  - **Framework:** Spring AI 2.0.x.
  - **Local Models:** Ollama running `llama3.1:latest` for natural language inference and `all-minilm:latest` for vector embeddings.
  - **Vector Storage:** PostgreSQL with the `pgvector` extension utilizing HNSW indexing and Cosine distance similarity.
  - **Model Context Protocol (MCP) Tools:** Native Spring AI callbacks with strict deduplication:
    * `CatalogTools.semanticProductSearch`: Real-time semantic catalog retrieval.
    * `CustomerTools.getCustomerHistory`: Equipment maintenance tracking and expired hydro-static test lookup by RIF/CNPJ.
    * `DocumentTools.searchInManuals`: Normative safety standard lookup (NFPA 10, COVENIN).
  - **Reactive Streaming:** `Flux<String>` with SSE protocol preventing local inference timeouts.

* **Database & Infrastructure:**
  - **Relational & Vector Database:** PostgreSQL 16 (`fire_safety_db`) running via Docker Compose on port `5433`.
  - **DevOps & CI/CD:** Decoupled GitHub Actions workflows (`ci-frontend.yml` and `backend-ci.yml`) automating build and test validation.
  - **Hosting:** Vercel Edge Network for Frontend and AWS Cloud for Backend.

---

## 2. Repository & Deployment Links

| Component | Repository Link | Live Deployment | Status |
| :--- | :--- | :--- | :--- |
| **Frontend Web App** | [![GitHub](https://img.shields.io/badge/GitHub-frontend--extin-181717?logo=github)](https://github.com/jhonduarte7/frontend-extin) | [venefast.com](https://venefast.com) | ![Active](https://img.shields.io/badge/Status-Production%20Active-brightgreen) |
| **Backend REST & RAG API** | [![GitHub](https://img.shields.io/badge/GitHub-backend--extin-181717?logo=github)](https://github.com/jhonduarte7/backend-extin) | [AWS Cloud](https://github.com/jhonduarte7/backend-extin) | ![Active](https://img.shields.io/badge/Status-Cloud%20Deployed-blue) |

---

## 3. Key Features & Repository Structure

### 3.1 Key Functional Modules

1. **Fiscal Invoicing & Physical Slip Replication:**
   - Exact digital reproduction of Equi-Extin's official physical paper invoice/order book, featuring fiscal control serials (`00-00418X`), laser/thermal printer formatting, and SENIAT-compliant layouts.
   - Dual tax mode configuration: **Con IVA** (Base Imponible + 16% Tax breakdown) vs. **Sin IVA** (Contado / simple delivery note).
   - Real-time conversion between **USD** reference base and official Venezuelan Bolívares (**VES**) via official BCV exchange rate integration.

2. **AI Technical Copilot & Document RAG:**
   - Normative document ingestion supporting PDF uploads (NFPA 10, COVENIN 1040) with automatic chunking and vector indexing in `pgvector`.
   - Real-time Server-Sent Events (SSE) streaming delivering token-by-token rendering with dynamic tool-calling visualizers.
   - Fault-tolerant resilience with 120s/180s HTTP timeout handling and automatic Markdown "Modo Contingencia" fallback responses.

3. **Customer Fleet & Workshop Maintenance (CRM):**
   - Comprehensive customer directory with Venezuelan Tax IDs (RIF `J-`, `V-`, `G-`) and Brazilian CNPJs.
   - Preventive maintenance alerts highlighting expired 5-year and 12-year hydro-static testing (PH) schedules in corporate red (`#E53935`).
   - Workshop inventory tracking for cylinders, chemical agents (PQS, $CO_2$, Solkaflam), valves, and security seals.

4. **Professional Engineering Standards:**
   - Built under **Specification-Driven Development (SDD)**, where requirements are rigorously documented in `docs/` before implementation.
   - Standardized `AGENT.md` / `AGENTS.md` engineering guides for AI coding agents.
   - Unified global pagination (strictly 5 items per page) across all administrative grids.
   - Automated continuous integration pipelines for both frontend and backend verifying builds on every `push` to `main`.

### 3.2 Workspace Directory Layout

```text
Equi-Extin/
├── backend-extin/                 # Spring Boot 3.x / Java 21 Backend Service
│   ├── .github/workflows/         # GitHub Actions CI (backend-ci.yml)
│   ├── src/main/java/             # Controllers, Services, Mappers, MCP Tools, JPA Entities
│   ├── src/test/java/             # 90 Automated Unit & Integration Tests (100% passing)
│   ├── docker-compose.yml         # PostgreSQL 16 + pgvector container configuration
│   ├── pom.xml                    # Maven configuration targeting Java 21
│   ├── AGENTS.md                  # Backend AI Engineering & Architectural Standards
│   └── README.md                  # Comprehensive Backend Technical Documentation
│
├── frontend-extin/                # React 19 + Vite 8 B2B Store & ERP Admin Frontend
│   ├── .github/workflows/         # GitHub Actions CI (ci-frontend.yml)
│   ├── src/components/            # UI Primitives, Checkout Drawer, Physical Slip Modals
│   ├── src/pages/                 # B2B Storefront, Catalog, Admin ERP Pages, RAG Chat
│   ├── src/locales/               # Trilingual i18n dictionaries (ES, EN, PT)
│   ├── package.json               # Node.js dependencies and scripts
│   ├── AGENT.md                   # Frontend AI Engineering & Architectural Standards
│   └── README.md                  # Comprehensive Frontend Technical Documentation
│
├── docs/                          # Single Source of Truth (SDD Specifications)
│   ├── api-contracts.md           # Standardized REST endpoint JSON contracts
│   ├── frontend-spec.md           # UI/UX design tokens, layout blueprints, and flows
│   └── system-spec.md             # Data model, relational core, and AI specifications
│
├── .gitignore                     # Workspace root hygiene exclusions
└── README.md                      # Unified Root Master Documentation
```

### 3.3 Repository Hygiene & `.gitignore` Exclusions

Both repositories and the workspace root maintain strict version control cleanliness. The following file categories are systematically excluded via `.gitignore`:
- **Build Artifacts & Runtimes:** `target/`, `dist/`, `build/`, `*.jar`, `*.war`, `node_modules/`.
- **IDE Configurations & Editor Caches:** `.idea/`, `.vscode/`, `*.iml`, `*.iws`, `.settings/`, `.classpath/`.
- **Runtime Logs & Temporary Session Dumps:** `*.log`, `npm-debug.log*`, `yarn-error.log*`, `cookies.txt`.
- **Local Scratch & Ephemeral Notes:** Scratch scripts, local tracking files (`QUESTION.md`, internal trajectory dumps).

---

## 4. Author & Engineering Leadership

Designed, architected, and developed under enterprise software engineering practices by:

**Jhonn Duarte**  
*Senior Software Architect & Full-Stack Engineer*  
GitHub: [@jhonduarte7](https://github.com/jhonduarte7)  
Live Production Platform: [venefast.com](https://venefast.com)

---

## License

Proprietary enterprise software developed for **Equi-Extin**. All rights reserved.
