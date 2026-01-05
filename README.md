# iKube 2.0 DevOps Portal integration

The iKube 2.0 DoP Integration is a middleware system designed to enable self-service provisioning and management of Kubernetes projects within the Swisscom DevOps Portal (DoP). It acts as a service broker that translates user intents (creating, managing, or deleting a project) into concrete infrastructure actions performed by the Swisscom Cloud Orchestrator (SCO).


Architectural Decisions :

The system follows a Serverless, Event-Driven Architecture

*Serverless:* It relies entirely on AWS managed services (Lambda, API Gateway, DynamoDB, Step Functions), eliminating the need for server management.
*Event-Driven & Asynchronous:* Long-running operations (provisioning, activation, termination) are decoupled from the user interface. User actions update a state in the database (DynamoDB), which triggers asynchronous workflows via streams to perform the actual infrastructure work.

The system is organized into three primary architectural blocks that manage the lifecycle of a project request from the UI to the cloud provider. 

### Major Architectural Blocks
*Frontend (Integration Module):* A set of UI components and pages (Landing, Create, Delete) that provide the user interface inside the DevOps Portal.

*Backend Services (Lambda functions):* Discrete functions that handle specific operations like lambda-create-k8s-project, lambda-read-catalog, and lambda-read-service-instance.

*Infrastructure & Shared Layers:* AWS CDK-defined resources and Lambda Layers that provide shared services like Database access (DynamoDB), Secret Management, and API communication.
