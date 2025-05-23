# Model Management

## 1. Feature Overview

The Model Management feature provides a unified interface for managing and accessing various Large LanguageModels (LLMs), Text Embedding models, Rerank models, Speech-to-Text, Moderation, and Text-to-Speech models from different providers. It supports configuring credentials for these providers, setting up specific models with their parameters, managing quotas, and enabling/disabling models for use within a workspace. This allows applications to flexibly leverage a diverse set of AI capabilities.

## 2. Architecture Diagram

```mermaid
flowchart TD
    subgraph AppWorkflow [Application / Workflow Configuration]
        A[App Model Config / Workflow Node] --> B{Model Selection Request}
    end

    subgraph ModelAccess [Model Access & Invocation]
        B --> C[ModelManager]
        C -- Gets Provider Config --> ProviderMgr[ProviderManager]
        ProviderMgr -- Fetches DB Config --> ProviderDB[(Provider/ProviderModel Tables)]
        C -- Returns --> MI[ModelInstance]
        MI -- Invokes --> ModelTypeIF[ModelType Interface (e.g., LargeLanguageModel)]
    end

    subgraph ModelRuntimeProviders [Model Runtime & Providers]
        ModelTypeIF -- Uses Factory for Provider Logic --> MPF[core/model_runtime/model_providers/ModelProviderFactory]
        MPF -- Gets Provider Schema --> ProviderPlugin[Plugin-based Provider Schema/Logic]
        ProviderPlugin -- (If applicable) Interacts with --> ExternalAPI[External Model API (e.g., OpenAI, Anthropic)]
        LocalOrCustomProvider[Custom/Local Provider Logic] -- (If applicable) Interacts with --> LocalEndpoint[Local Model Endpoint]
        MPF -- Returns Model Schema/Instance --> ModelTypeIF
    end

    subgraph ProviderAdminConfig [Provider Configuration - Console/API]
        N[Console UI for Providers] --> O{API Request}
        O --> P["/console/workspace/model-providers"]
        P --> ModelProviderSvc[ModelProviderService]
        ModelProviderSvc -- CRUD Operations --> ProviderDB
        ModelProviderSvc -- Uses --> ProviderMgr
    end
```

**Diagram Explanation:**

*   **Application / Workflow Configuration:** Applications or workflow nodes specify the model they need (e.g., by provider name, model name, model type).
*   **Model Access & Invocation:**
    *   `ModelManager` is the primary entry point for obtaining a runnable `ModelInstance`.
    *   `ProviderManager` is used by `ModelManager` to fetch the configuration (including credentials) for the requested provider from the database (`Provider` and `ProviderModel` tables).
    *   A `ModelInstance` is created, encapsulating the model's configuration, credentials, and a reference to the appropriate `ModelType` interface implementation (e.g., `LargeLanguageModel`).
    *   When a method like `invoke_llm()` is called on `ModelInstance`, it delegates the call to the specific `ModelType` interface.
*   **Model Runtime & Providers:**
    *   The `ModelProviderFactory` is responsible for providing the schema and logic for different model providers. For plugin-based providers (which seems to be the primary architecture), it interacts with the plugin system to get provider declarations, model lists, and potentially the execution logic itself or a client to interact with a plugin-hosted model.
    *   The actual model provider logic (whether built-in or via plugin) then interacts with the external model API (e.g., OpenAI) or a local model endpoint.
*   **Provider Configuration - Console/API:**
    *   The Console UI allows administrators to manage model providers.
    *   API endpoints (under `/console/workspace/model-providers` and `/console/workspace/models`) handle requests for configuring providers, setting credentials, enabling/disabling models, and setting default models.
    *   `ModelProviderService` contains the business logic for these operations, interacting with `ProviderManager` and directly with the database models (`Provider`, `ProviderModel`, `TenantDefaultModel`).

## 3. Related Modules

| Path                                                              | Role                                                                                  | Major Classes/Functions                                                                |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `api/core/model_manager.py`                                       | Manages model instances, configurations, and load balancing.                          | `ModelManager`, `ModelInstance`, `LBModelManager`                                        |
| `api/core/provider_manager.py`                                    | Manages provider configurations and access to provider details.                       | `ProviderManager` (Used by `ModelManager` and `ModelProviderService`)                  |
| `api/core/model_runtime/model_providers/model_provider_factory.py`| Discovers and provides access to model provider implementations (mainly plugin-based).| `ModelProviderFactory`, `get_provider_schema`, `get_model_type_instance`               |
| `api/core/model_runtime/entities/model_entities.py`               | Pydantic models for AI models, their properties, features, and parameters.            | `ModelType`, `AIModelEntity`, `ParameterRule`, `ModelFeature`                          |
| `api/core/model_runtime/entities/provider_entities.py`            | Pydantic models for provider schemas, credentials, and configuration forms.           | `ProviderEntity`, `CredentialFormSchema`, `ProviderConfig`                             |
| `api/core/model_runtime/model_providers/__base__/`                | Abstract base classes for different model types (LLM, Embedding, Rerank, etc.).       | `LargeLanguageModel`, `TextEmbeddingModel`, `RerankModel`, `ModerationModel`, `TTSModel`, `Speech2TextModel` |
| `api/services/model_provider_service.py`                          | Business logic for managing provider configurations, credentials, and default models. | `ModelProviderService`                                                                 |
| `api/controllers/console/workspace/model_providers.py`            | API endpoints for managing provider-level configurations and credentials.               | `ModelProviderListApi`, `ModelProviderCredentialApi`, `ModelProviderApi`               |
| `api/controllers/console/workspace/models.py`                     | API endpoints for managing specific models, their credentials, and defaults.          | `DefaultModelApi`, `ModelProviderModelApi`, `ModelProviderModelEnableApi`              |
| `api/models/provider.py`                                          | SQLAlchemy database models for storing provider configurations and credentials.       | `Provider`, `ProviderModel`, `TenantDefaultModel`, `ProviderModelSetting`, `LoadBalancingModelConfig` |

## 4. Data Flow (Typical LLM Call)

1.  **Input:** An application component (e.g., `AgentRunner`, Workflow `LLMNode`, or direct API call) requires an LLM. It specifies the `tenant_id`, `provider` name (e.g., "openai"), `model_type` (e.g., `ModelType.LLM`), and `model` name (e.g., "gpt-3.5-turbo"). It also provides the prompt and any specific model parameters.
2.  **Model Instance Retrieval (`ModelManager`):**
    *   The component calls `ModelManager.get_model_instance(tenant_id, provider, ModelType.LLM, model_name)`.
    *   `ModelManager` uses `ProviderManager` to fetch the `ProviderModelBundle`.
    *   `ProviderManager` queries the `Provider` and `ProviderModel` tables (from `api/models/provider.py`) for the given `tenant_id` and `provider_name` to get stored configurations, including encrypted credentials. It also uses `ModelProviderFactory` to get the provider's schema (supported models, parameter rules, etc.).
    *   The `ModelProviderFactory` (in `api/core/model_runtime/model_providers/model_provider_factory.py`) provides the specific `LargeLanguageModel` interface implementation for that provider (likely a plugin client).
    *   A `ModelInstance` is created, containing the provider's schema, the specific model's schema, decrypted credentials, and the `LargeLanguageModel` interface.
3.  **Invocation (`ModelInstance`):**
    *   The component calls `model_instance.invoke_llm(prompt_messages, model_parameters, stream=True/False, user=user_id)`.
    *   If load balancing is enabled (for custom providers), `LBModelManager` selects credentials using a round-robin strategy, potentially respecting cooldown periods for failing endpoints (managed via Redis).
    *   The `invoke()` method of the `LargeLanguageModel` interface (which points to the actual provider's client/logic, often via the plugin system) is called.
4.  **Provider-Specific Logic (Plugin or Built-in):**
    *   The specific provider's implementation (e.g., an OpenAI plugin or a direct OpenAI client) formats the request according to that provider's API specifications.
    *   It makes an HTTP request to the external API endpoint (e.g., `https://api.openai.com/v1/chat/completions`), including the API key from the resolved credentials.
    *   It receives the response from the external API (either as a full response or a stream).
    *   It parses this response, converting it into Dify's standardized `LLMResult` or a generator of `LLMResultChunk` objects. This includes mapping response fields, extracting token usage, and calculating costs based on the model's pricing information.
    *   Provider-specific errors are caught and potentially mapped to Dify's common invocation error types.
5.  **Output:** The `LLMResult` (or generator of chunks) is returned up the call stack to the initial application component.

## 5. Extension Points

*   **Adding a New Model Provider (Plugin-based):**
    1.  Develop a new plugin that conforms to Dify's model provider plugin interface. This involves:
        *   Defining a `PluginModelProviderEntity` which includes the provider's identity (name, label, icon), supported model types, a list of `AIModelEntity` for its models, and schemas for provider/model credentials (`ProviderCredentialSchema`, `ModelCredentialSchema`).
        *   Implementing the server-side logic for the plugin to handle requests (e.g., list models, validate credentials, invoke models by calling the actual external API). This logic would be exposed via endpoints defined in the plugin's manifest.
    2.  The `ModelProviderFactory` dynamically discovers registered and enabled plugins.
    3.  Once a plugin provider is active, users can configure its credentials through the Dify console UI (`ModelProviderService` and controllers handle saving this to the `Provider` table).
*   **Adding a New Model to an Existing Provider (Plugin-based):**
    1.  If the provider plugin fetches its model list dynamically from the provider's API, new models might appear automatically after the plugin is updated or Dify's cache is refreshed.
    2.  If the plugin hardcodes its model list, the plugin itself needs to be updated to include the new `AIModelEntity` in its declaration.
*   **Customizing Model Parameters:**
    *   The `ParameterRule` definitions within `AIModelEntity` (part of a provider's schema) dictate which parameters are available for a model and how they are configured (e.g., `temperature`, `max_tokens`). Plugins can define these rules for their models.
    *   Applications can then pass these parameters during invocation, and they will be sent to the underlying model API by the provider's implementation.
*   **Load Balancing for Custom Models:**
    *   For providers of type "custom", multiple sets of credentials or endpoints can be configured via the `LoadBalancingModelConfig` table, managed through `ModelLoadBalancingService` and the UI. `LBModelManager` handles the selection during runtime.
```
