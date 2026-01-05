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
    %% === Frontend Layer ===
    subgraph Frontend_Layer["Frontend Layer: dop-integration-ikube2<br>Lit, @swisscom/sdx, TypeScript<br>Handles user input for Kubernetes projects"]
        style Frontend_Layer fill:#FFDDC1,stroke:#FF6600,stroke-width:2px
        UI[DevOps Portal UI]
        style UI fill:#FFEECC,stroke:#FF6600
    end

    %% === API Layer ===
    subgraph API_Layer["API Layer - Backend: API Gateway & Lambda<br>Node.js 22.x, OSB API v2.16<br>Exposes OSB endpoints, validates, checks compliance, authenticates"]
        style API_Layer fill:#C1E1FF,stroke:#0066CC,stroke-width:2px
        APIGW[API Gateway]
        Auth[Lambda Authorizer]
        L_Create[Lambda: Create/Update]
        L_Read[Lambda: Read/Catalog]
        style APIGW fill:#CCE5FF,stroke:#0066CC
        style Auth fill:#CCE5FF,stroke:#0066CC
        style L_Create fill:#CCE5FF,stroke:#0066CC
        style L_Read fill:#CCE5FF,stroke:#0066CC
    end

    %% === Data Layer ===
    subgraph Data_Layer["Data Layer: DynamoDB<br>Single-table design, source of truth, emits events via Streams"]
        style Data_Layer fill:#D1FFC1,stroke:#33AA00,stroke-width:2px
        DDB[(DynamoDB: Service Instances)]
        Stream[DynamoDB Stream]
        style DDB fill:#E6FFCC,stroke:#33AA00
        style Stream fill:#E6FFCC,stroke:#33AA00
    end

    %% === Orchestration Layer ===
    subgraph Orchestration_Layer["Orchestration Layer: Step Functions & EventBridge Pipes<br>Coordinates lifecycle workflows (Provisioning, Termination)"]
        style Orchestration_Layer fill:#FFD1F6,stroke:#CC0099,stroke-width:2px
        Pipe[EventBridge Pipe]
        SFN[Step Function: Lifecycle Engine]
        subgraph Workflow_Engines
            Prov[Provisioning Engine]
            Term[Termination Engine]
            style Prov fill:#FFCCEE,stroke:#CC0099
            style Term fill:#FFCCEE,stroke:#CC0099
        end
        L_SCO[Lambda: SCO Client]
        style Pipe fill:#FFCCEE,stroke:#CC0099
        style SFN fill:#FFCCEE,stroke:#CC0099
        style L_SCO fill:#FFCCEE,stroke:#CC0099
    end

    %% === External Systems ===
    subgraph External_Systems["External Systems"]
        style External_Systems fill:#F0F0F0,stroke:#888888,stroke-width:2px
        SCO_API[Swisscom Cloud Orchestrator]
        PUG_API[Platform Usage Guardrails]
        style SCO_API fill:#FFFFFF,stroke:#888888
        style PUG_API fill:#FFFFFF,stroke:#888888
    end

    %% === Flows ===
    UI -->|OSB API Requests| APIGW
    APIGW -->|Auth Check| Auth
    APIGW -->|Route| L_Create
    APIGW -->|Route| L_Read
    
    L_Create -->|Validate & Check| PUG_API
    L_Create -->|Persist State| DDB
    
    DDB -->|Change Event| Stream
    Stream -->|Filter & Route| Pipe
    Pipe -->|Trigger| SFN
    
    SFN -->|Route Logic| Prov
    SFN -->|Route Logic| Term
    
    Prov -->|Execute Task| L_SCO
    Term -->|Execute Task| L_SCO
    
    L_SCO -->|REST API| SCO_API

  ```
test test test






