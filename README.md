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

  To provide a clear, technical view of how the system handles a request from start to finish, the following sequence diagram summurise the "Create Project" flow across all four architectural layers.

```mermaid
sequenceDiagram
    autonumber
    
    box "Frontend Layer" #dae8fc
        participant User
        participant FE as Web Component
    end

    box "API Layer (Backend)" #d5e8d4
        participant APIGW as API Gateway
        participant LAM_CREATE as Lambda<br/>(Create Service Instance)
    end

    box "Data Layer" #ffe6cc
        participant DDB as DynamoDB
        participant STREAM as DynamoDB Stream
    end

    box "Orchestration Layer" #e1d5e7
        participant PIPE as EventBridge Pipe
        participant SFN as Step Function
        participant LAM_SCO as Lambda (SCO Client)
    end

    User->>FE: Click "Create Project"
    FE->>APIGW: PUT /v2/service_instances/:id
    APIGW->>LAM_CREATE: Invoke
    
    activate LAM_CREATE
    Note right of LAM_CREATE: Validate Identity & PUG Compliance
    LAM_CREATE->>DDB: PutItem (State: READY)
    LAM_CREATE-->>APIGW: 202 Accepted
    deactivate LAM_CREATE
    
    APIGW-->>FE: 202 Accepted
    
    DDB->>STREAM: Item Change Event
    STREAM->>PIPE: Filter (State == READY)
    PIPE->>SFN: Trigger Provisioning Workflow
    
    activate SFN
    SFN->>LAM_SCO: Create Project in Kubermatic
    LAM_SCO-->>SFN: Success
    SFN->>DDB: Update State to ACTIVE
    deactivate SFN
  ```

The Frontend Layer: The User Interface

   1. Overview
      
The frontend is a client-side application implemented with Native Web Components and delivered as a single JavaScript bundle built via Rollup, which is loaded dynamically by the Swisscom DevOps Portal at runtime. Running entirely in the user’s browser, it communicates directly with the API layer and follows a Single Page Application (SPA) architecture. Its primary responsibilities include handling user interactions, performing form validation, and presenting the Service Catalog in a responsive and user-friendly manner.

   2. Architecture & Key Concepts

This frontend is not a standalone website but a micro-frontend plugin that lives inside the Swisscom DevOps Portal.Here is the high-level map of how the frontend is wired. 

```mermaid
graph LR
    %% -- Styles --
    classDef host fill:#f9fafb,stroke:#6b7280,stroke-width:2px,stroke-dasharray: 4 2,color:#374151,stroke-linecap:round,stroke-radius:10px;
    classDef plugin fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0369a1,stroke-linecap:round,stroke-radius:10px;
    classDef component fill:#ffffff,stroke:#0284c7,stroke-width:1.5px,color:#111827,stroke-linecap:round,stroke-radius:8px,stroke-shadow:2px 2px 6px #cbd5e1;
    classDef service fill:#fff7ed,stroke:#f97316,stroke-width:1.5px,color:#1f2937,stroke-linecap:round,stroke-radius:8px,stroke-shadow:2px 2px 6px #fde8dc;
    classDef cloud fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d,stroke-linecap:round,stroke-radius:10px,stroke-shadow:2px 2px 6px #a7f3d0;

    %% -- The Browser Environment --
    subgraph Browser_Window [User Browser]
        direction TB

        %% 1. The Host App
        subgraph DOP_Host [Swisscom DevOps Portal]
            direction TB
            Host_Nav[Global Navigation]
            Host_Context[User Context]
        end

        %% 2. Our Plugin
        subgraph Our_Plugin [iKube Plugin Container]
            direction TB
            
            %% Entry Point
            Entry[Entry Component]:::component

            %% The Logic Core
            subgraph Core_Logic [Core Logic]
                Router[Router Service]:::service
                Services[Backend Service]:::service
                Bridge[DOP Service]:::service
            end

            %% The UI Pages
            subgraph Views [UI Pages]
                Page_List[Landing Page]:::component
                Page_Create[Create Wizard]:::component
            end
        end
    end

    %% -- The Outside World --
    subgraph AWS_Cloud [AWS Backend]
        API_GW[API Gateway]:::cloud
    end

    %% -- Wiring It All Together --
    
    %% Host loads Plugin
    DOP_Host -->|1. Load Script| Entry
    
    %% Plugin Internal Flow
    Entry -->|2. Boot| Router
    Router -->|3. Render| Page_List
    Router -->|3. Render| Page_Create
    
    %% Data Fetching
    Page_List -->|Fetch| Services
    Services == 4. HTTPS Request ==> API_GW
    
    %% Talking back to Host
    Page_Create -->|Notify| Bridge
    Bridge -.->|5. PostMessage| DOP_Host

    %% Apply Styles
    class DOP_Host host;
    class Our_Plugin plugin;
    class AWS_Cloud cloud;
  ```




