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
Technology Stack :

Frontend: Lit (Web Components), TypeScript, RXJS, @swisscom/sdx (Swisscom UI kit).

Backend: Node.js 22.x, AWS Lambda, Amazon API Gateway.

Orchestration: AWS Step Functions (Standard Workflows), EventBridge Pipes.

Infrastructure: AWS CDK (Cloud Development Kit) in TypeScript.






