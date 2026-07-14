# Marketing Event Delivery Report — Flow Diagrams

## Flow 01：查詢、權限與輸出

```mermaid
flowchart TD
    A[Open Marketing Event Delivery Report] --> B{Authenticated BO role}
    B -->|Admin| C[Scope: all agents]
    B -->|Agent| D[Scope: authenticated agent_id]
    C --> E[Apply defaults: Status SUCCESS and current BO day]
    D --> E
    E --> F[Submit filters]
    F --> G{Date range valid and no more than 31 days?}
    G -->|No| H[Show validation error and do not query]
    G -->|Yes| I[GET marketing event deliveries]
    I --> J[Backend applies role scope and reads RD delivery log]
    J --> K[Mask sensitive values]
    K --> L{Records found?}
    L -->|No| M[Show No Data]
    L -->|Yes| N[Show 10 records per page ordered newest first]
    N --> O{User action}
    O -->|View details| P[GET delivery by delivery_id]
    P --> Q{Record inside role scope?}
    Q -->|No| R[Return 403]
    Q -->|Yes| S[Show masked request and provider response]
    O -->|Export CSV| T[Export with the same filters and role scope]
    T --> U[Exclude PII, request body, and response body]
```

