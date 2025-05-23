# Platform Architecture Overview

This document provides a high-level overview of the Dify platform architecture. Dify is an LLM application development platform that enables users to create, manage, and deploy AI-powered applications. Its architecture is composed of a user-facing Console (web application), a set of APIs for programmatic access and external integrations, and a suite of Core Services that power the platform's functionalities.

## Core Features

The platform is built around the following core features, each detailed in their respective documents:

*   [User Authentication](./feature-user-authentication.md)
*   [Application Management](./feature-application-management.md)
*   [Dataset Management](./feature-dataset-management.md)
*   [Workflow Engine](./feature-workflow-engine.md)
*   [Agent Framework](./feature-agent-framework.md)
*   [Model Management](./feature-model-management.md)
*   [Plugin System](./feature-plugin-system.md)

## Shared Resources

Commonly used terms and definitions can be found in the glossary:

*   [Glossary](./_shared/glossary.md)

## High-Level System Diagram

The following diagram illustrates the interaction between the main components of the Dify platform:

```mermaid
graph TD
    UserConsole[User via Console UI] --> APILayer[API Layer (Flask)]
    DeveloperAPI[Developer via External API] --> APILayer

    APILayer --> AuthSvc[User Authentication Service]
    APILayer --> AppSvc[Application Service]
    APILayer --> DatasetSvc[Dataset Service]
    APILayer --> WorkflowService[Workflow Service]
    APILayer --> ModelProviderSvc[Model Provider Service]
    APILayer --> PluginSvc[Plugin Service]
    APILayer --> AgentSvc[Agent Service (Logs)]

    AppSvc --> ModelConfigSvc[App Model Config Service]
    AppSvc --> WorkflowService
    AppSvc --> AgentFramework[Agent Framework]
    AppSvc --> ModelRuntime[Model Runtime]
    AppSvc --> DatasetSvc

    WorkflowService --> GraphEngine[Graph Engine]
    GraphEngine --> NodeImplementations[Workflow Nodes (LLM, Tools, Logic)]
    NodeImplementations --> ModelRuntime
    NodeImplementations --> ToolMgr[Tool Manager]
    NodeImplementations --> DatasetSvc

    AgentFramework --> ModelRuntime
    AgentFramework --> ToolMgr
    
    DatasetSvc --> RAGPipeline[RAG Pipeline (Extract, Split, Embed)]
    RAGPipeline --> ModelRuntimeTextEmbedding[Model Runtime (Text Embedding)]
    RAGPipeline --> VectorDB[(Vector Database)]

    ToolMgr --> BuiltinTools[Built-in Tools]
    ToolMgr --> APITools[API Tools (Custom)]
    ToolMgr --> WorkflowTools[Workflow-as-Tool]
    ToolMgr --> PluginTools[Plugin Tools]
    PluginTools -- via PluginService --> PluginDaemon[Plugin Daemon]

    ModelProviderSvc --> ModelMgr[Model Manager]
    ModelMgr --> ModelRuntime

    AuthSvc --> Database[(Core Database - PostgreSQL)]
    AppSvc --> Database
    ModelConfigSvc --> Database
    DatasetSvc --> Database
    WorkflowService --> Database
    ModelProviderSvc --> Database
    AgentSvc --> Database
    PluginSvc -- (for API-Based Ext.) --> Database
```
