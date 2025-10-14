## Контекстная диаграмма (Mermaid)

```mermaid
flowchart LR
  %% Trust Boundaries
  subgraph TB1["Client Boundary"]
    U["User / Client"]
  end

  subgraph TB2["Edge Boundary"]
    G["Reverse Proxy / API Gateway"]
  end

  subgraph TB3["Core Boundary"]
    A["FastAPI App (main.py)"]
    RL["ReadingList Service"]
  end

  subgraph TB4["Data Boundary"]
    S["Storage (In-Memory / DB)"]
    L["Logs / Metrics"]
  end

  %% Flows
  U -- "F1: HTTPS GET/POST" --> G
  G -- "F2: HTTP (internal)" --> A
  A -- "F3: Internal call" --> RL
  RL -- "F4: Read/Write" --> S
  A -- "F5: Errors/Events" --> L
  U -- "F6: CORS Preflight" --> G
  U -- "F7: Auth Headers/JWT?" --> G
  A -- "F8: Response + Security Headers" --> U
