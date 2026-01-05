# iKube 2.0 DevOps Portal integration

The iKube 2.0 DevOps Portal Integration project serves as the bridge between the Swisscom DevOps Portal (DOP) and the iKube 2.0 Kubernetes platform (powered by Kubermatic). Its primary purpose is to enable self-service capabilities for application owners to provision, manage, and decommission Kubernetes projects directly through the DOP interface.

the  main goals of this integration is : 
    - Automation: Automate the lifecycle management of Kubernetes projects, replacing manual fulfillment processes.
    - Standardization: Enforce compliance and configurations like IAM roles and network settings during project creation.


With this integration, we removed manual Kubernetes project provisioning, enforced policy-driven compliance, automated identity-to-role mapping, and streamlined long-running lifecycle operations.

Architectural Overview :

  The iKube 2.0 DevOps Portal Integration is built on a Serverless Event-Driven Architecture. The system is organized into four distinct logical layers, ensuring separation of concerns between user interaction, API handling, state persistence, and complex workflow orchestration.


  ```mermaid
graph TD
    %% Global Styles
    classDef blackText fill:#fff,stroke:#333,stroke-width:2px,color:#000;

    subgraph Frontend_Layer [Frontend Layer]
        UI["<b>UI</b><br/>Handles user input &<br/>manages K8s project requests"]:::blackText
    end

    subgraph API_Layer [API Layer - Backend]
        direction TB
        APIGW["<b>API Gateway</b><br/>Exposes OSB endpoints"]:::blackText
        Auth["<b>Security Authorizer</b><br/>Verifies DOP security tokens"]:::blackText
        L_Create["<b>Lambda Handlers</b><br/>Create, Read Catalog, List"]:::blackText
    end

    subgraph Data_Layer [Data Layer]
        DDB[("<b>DynamoDB</b><br/>Source of truth")]:::blackText
        Stream["<b>DynamoDB Stream</b><br/>Captures state changes to drive<br/>async workflows"]:::blackText
    end

    subgraph Orchestration_Layer [Orchestration Layer]
        Pipe["<b>EventBridge Pipe</b><br/>Filters events and triggers<br/>the Lifecycle Engine"]:::blackText
        SFN["<b>Step Functions</b><br/>Manages long-running Provisioning,<br/>Activation, & Termination workflows"]:::blackText
        L_SCO["<b>SCO Client</b><br/>Communicating<br/>with SCO"]:::blackText
    end

    subgraph External_Systems [External Systems]
        SCO_API["<b>SCO API</b><br/>Kubermatic Project Management"]:::blackText
        PUG_API["<b>PUG API</b><br/>Guardrails & Compliance"]:::blackText
    end

    %% Flow Connections
    UI -->|OSB REST API| APIGW
    APIGW -.-> Auth
    APIGW --> L_Create
    
    L_Create -->|Compliance Check| PUG_API
    L_Create -->|Save READY state| DDB
    
    DDB --> Stream
    Stream --> Pipe
    Pipe -->|Trigger| SFN
    
    SFN -->|Task Execution| L_SCO
    L_SCO -->|REST API| SCO_API

    %% Layer Coloring
    style Frontend_Layer fill:#dae8fc,stroke:#6c8ebf,color:#000
    style API_Layer fill:#d5e8d4,stroke:#82b366,color:#000
    style Data_Layer fill:#ffe6cc,stroke:#d79b00,color:#000
    style Orchestration_Layer fill:#e1d5e7,stroke:#9673a6,color:#000
    style External_Systems fill:#f5f5f5,stroke:#666666,color:#000
  ```

  To provide a clear, technical view of how the system handles a request from start to finish, the following sequence diagram traces the "Create Project" flow across all four architectural layers.

```mermaid
sequenceDiagram
    autonumber
    
    box "Frontend Layer" #dae8fc
        participant User
        participant FE as Web Component<br/>(dop-integration-ikube2)
    end

    box "API Layer (Backend)" #d5e8d4
        participant APIGW as API Gateway
        participant AUTH as Lambda Authorizer
        participant LAM_CREATE as Lambda<br/>(Create Service Instance)
    end

    box "Data Layer" #ffe6cc
        participant DDB as DynamoDB<br/>(Service Instances)
        participant STREAM as DynamoDB Stream
    end

    box "Orchestration Layer" #e1d5e7
        participant PIPE as EventBridge Pipe
        participant SFN as Step Function<br/>(Lifecycle Engine)
        participant LAM_SCO as Lambda<br/>(SCO Client)
    end

    box "External Systems" #f5f5f5
        participant PUG as PUG API<br/>(Compliance)
        participant SCO as SCO API<br/>(Orchestrator)
    end

    %% -- SYNC PHASE --
    Note over User, DDB: Phase 1: Request Ingestion (Synchronous)
    
    User->>FE: Fill Form & Click "Create Project"
    FE->>APIGW: PUT /v2/service_instances/:instance_id
    
    APIGW->>AUTH: Authenticate Request
    AUTH-->>APIGW: Allow
    
    APIGW->>LAM_CREATE: Invoke Handler
    
    activate LAM_CREATE
    LAM_CREATE->>PUG: POST /check-compliance
    PUG-->>LAM_CREATE: Compliance OK
    
    LAM_CREATE->>DDB: PutItem (State: READY_FOR_PROVISIONING)
    activate DDB
    DDB-->>LAM_CREATE: Success
    deactivate DDB
    
    LAM_CREATE-->>APIGW: Response (202 Accepted)
    deactivate LAM_CREATE
    
    APIGW-->>FE: 202 Accepted
    FE-->>User: Show "READY_FOR_ACTIVATION"

    %% -- ASYNC PHASE --
    Note over DDB, SCO: Phase 2: Orchestration (Asynchronous)

    DDB->>STREAM: Capture "INSERT/MODIFY" Event
    activate STREAM
    STREAM->>PIPE: Filter Event (State == READY)
    deactivate STREAM
    
    activate PIPE
    PIPE->>SFN: Start Execution
    deactivate PIPE
    
    activate SFN
    SFN->>SFN: Determine Route (Provisioning)
    
    SFN->>LAM_SCO: Invoke: Create Project
    activate LAM_SCO
    LAM_SCO->>SCO: POST /projects (Create)
    SCO-->>LAM_SCO: Project ID Returned
    LAM_SCO-->>SFN: Task Success
    deactivate LAM_SCO
    
    loop Polling / Wait
        SFN->>LAM_SCO: Check Status
        LAM_SCO->>SCO: GET /projects/:id/status
        SCO-->>LAM_SCO: Status (e.g., Creating...)
        LAM_SCO-->>SFN: Status
    end
    
    SFN->>LAM_SCO: Final Configuration (IAM/Quotas)
    activate LAM_SCO
    LAM_SCO->>SCO: PUT /projects/:id (Config)
    deactivate LAM_SCO

    SFN->>DDB: UpdateItem (State: ACTIVE)
    deactivate SFN
    
    Note over User, DDB: Phase 3: Completion
    
    FE->>APIGW: GET /v2/service_instances/:last_operation
    APIGW->>DDB: Read State
    DDB-->>FE: State: ACTIVE
    FE-->>User: Show "Active" Dashboard

  ```


Technology Stack :

Frontend: Lit (Web Components), TypeScript, RXJS, @swisscom/sdx (Swisscom UI kit).

Backend: Node.js 22.x, AWS Lambda, Amazon API Gateway.

Orchestration: AWS Step Functions (Standard Workflows), EventBridge Pipes.

Infrastructure: AWS CDK (Cloud Development Kit) in TypeScript.


