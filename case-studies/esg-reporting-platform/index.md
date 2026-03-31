### Problem description

**Problem**: ESG (Environmental, Social, Governance) reporting and supplier engagement are fragmented across spreadsheets, emails, and ad‑hoc tools, making it hard for an “owner” organization to onboard clients, manage supplier accounts, and collect documentation consistently.

**Solution**: A two‑sided ESG platform with an **Owner Portal** for client onboarding and management (including authentication and client lifecycle) and a **Supplier Portal** for authenticated suppliers to log in, view a dashboard, and upload required ESG documents, backed by a shared API/auth layer and data store.

---

### Architecture diagram (mermaid)

```mermaid
flowchart TD
    subgraph browserLayer [Browser]
        OP["Owner Portal (React/TS)"]
        SP["Supplier Portal (React/TS + Supabase)"]
    end

    subgraph backendLayer [Backend]
        AUTH["Auth service (/auth - login, logout, me, refresh)"]
        CLIENTS["Clients API (/clients - CRUD clients, status)"]
    end

    subgraph infraLayer [Infrastructure]
        SUPA["Supabase Auth (users, sessions)"]
        DB["Postgres - ESG DB"]
        STORAGE["Object storage (documents, uploads)"]
    end

    OP -->|Bearer token via /auth| AUTH
    OP -->|Bearer token via /clients| CLIENTS

    SP -->|Supabase JS SDK - email+password| SUPA

    AUTH -->|validate user and issue JWT| SUPA
    AUTH -->|read/write auth users| DB
    CLIENTS -->|CRUD client records| DB
    SP -->|upload docs| STORAGE

    OP -.->|Owner creates client and gets temp password| DB
    DB -.->|Client user exists and can log in via Supplier Portal| SP
```

---

### UI screens (mermaid diagram)

```mermaid
flowchart LR
    subgraph OwnerPortal
        OLogin[Owner Login\nEmail + Password\nError state, loading]
        ODash[Owner Dashboard\nWelcome + user email]
        OClients[Clients List\nTable, status badges\nEdit/Delete actions]
        OClientModal[Client Form Modal\nCreate/Edit client\nCompany + Contact + Address + Business info]
    end

    subgraph SupplierPortal
        SLogin[Supplier Login\nSupabase auth]
        SDash[Supplier Dashboard\nWelcome text]
        SUPL[Uploads Page\nDropzone + Browse button]
    end

    OLogin --> ODash
    ODash --> OClients
    OClients -->|New Client| OClientModal
    OClientModal -->|Create/Update success| OClients

    SLogin --> SDash
    SDash --> SUPL
```

---

### Workflow (mermaid diagram)

```mermaid
sequenceDiagram
    participant Owner as Owner User (Owner Portal)
    participant OP as Owner Portal (React)
    participant AUTH as Auth Service (/auth)
    participant CLIENTS as Clients API (/clients)
    participant DB as ESG DB
    participant Supplier as Supplier User (Supplier Portal)
    participant SP as Supplier Portal (React+Supabase)
    participant SUPA as Supabase Auth

    Note over Owner,OP: Owner authentication
    Owner->>OP: Enter email + password
    OP->>AUTH: POST /auth/login
    AUTH->>SUPA: Verify credentials / create session
    SUPA-->>AUTH: Session (access + refresh token)
    AUTH-->>OP: AuthResponse (user + tokens)
    OP->>OP: Store tokens in localStorage
    OP-->>Owner: Redirect to /dashboard

    Note over Owner,CLIENTS: Client onboarding
    Owner->>OP: Navigate to Clients page
    OP->>CLIENTS: GET /clients?skip&limit (with Bearer token)
    CLIENTS->>DB: Fetch clients
    DB-->>CLIENTS: Client list
    CLIENTS-->>OP: JSON clients
    OP-->>Owner: Render clients table

    Owner->>OP: Click "New Client"
    OP-->>Owner: Show client modal form
    Owner->>OP: Submit client details
    OP->>CLIENTS: POST /clients/ (JSON body + Bearer token)
    CLIENTS->>DB: Insert client + generate temp password
    DB-->>CLIENTS: New client record (+ temp password)
    CLIENTS-->>OP: New client
    OP-->>Owner: Show success + temp password message

    Note over Supplier,SP: Supplier activation & uploads
    Supplier->>SP: Open Supplier Portal
    Supplier->>SP: Login with email + temp/new password
    SP->>SUPA: supabase.auth.signInWithPassword
    SUPA-->>SP: Session + user
    SP-->>Supplier: Supplier Dashboard

    Supplier->>SP: Go to Uploads
    SP-->>Supplier: Upload UI (dropzone/button)
    Supplier->>SP: Upload documents
    SP->>STORAGE: Store ESG docs (e.g. S3/Supabase storage)
    STORAGE-->>SP: Upload success
    SP-->>Supplier: Confirm success
```

*(You can use this sequence as a storyboard for a short workflow video.)*

---

### Results / impact

- **Operational efficiency**: Centralizes client onboarding and supplier access, reducing manual spreadsheet/email workflows and improving data consistency.
- **Data quality & traceability**: Structured client records and authenticated supplier uploads create a single source of truth for ESG‑related information and documents.
- **Security & access control**: Token‑based auth for owners and Supabase auth for suppliers ensure only authorized parties can see or modify ESG data.
- **Scalability**: Clear separation of frontends (owner/supplier), backend APIs, and storage enables incremental extension (e.g. more ESG metrics, reporting, analytics) without re‑architecting the core system.

