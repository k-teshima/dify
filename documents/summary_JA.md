# プラットフォームアーキテクチャ概要

このドキュメントは、Dify プラットフォームのアーキテクチャを概観します。Dify は LLM アプリケーション開発プラットフォームであり、ユーザーが AI 対応アプリケーションを作成、管理、展開できるようにします。アーキテクチャは、ユーザー向けのコンソール（Web アプリケーション）、プログラムによるアクセスや外部連携のための API 群、そしてプラットフォームの機能を支えるコアサービス群で構成されています。

## コア機能

プラットフォームは以下のコア機能を中心に構築されており、それぞれの詳細は個別のドキュメントで説明されています。

* [ユーザー認証](./feature-user-authentication.md)
* [アプリケーション管理](./feature-application-management.md)
* [データセット管理](./feature-dataset-management.md)
* [ワークフローエンジン](./feature-workflow-engine.md)
* [エージェントフレームワーク](./feature-agent-framework.md)
* [モデル管理](./feature-model-management.md)
* [プラグインシステム](./feature-plugin-system.md)

## 共有リソース

よく使用される用語や定義は以下の用語集にまとめられています。

* [用語集](./_shared/glossary.md)

## ハイレベルシステム図

以下の図は、Dify プラットフォームの主要コンポーネント間の相互作用を示しています。

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
