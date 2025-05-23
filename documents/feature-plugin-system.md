# Plugin System

## 1. Feature Overview

The Plugin System allows extending the platform's capabilities by integrating external tools, services, and models. Plugins can expose functionalities as tools (usable by agents and workflows), introduce new model providers, or add other extensions. The system is designed to be modular, relying on a "plugin daemon" for managing plugin installation, dependencies, and execution. This architecture enables a flexible way to enhance the platform's core features without modifying its main codebase directly.

## 2. Architecture Diagram

```mermaid
flowchart TD
    subgraph UserPluginManagement [Plugin Management - Console/API]
        A[User Installs/Manages Plugin via Console] --> B{API Request}
        B --> C["/console/extensions/plugins/*"]
        C --> PluginService[PluginService]
        PluginService -- Interacts with --> PluginDaemon[Plugin Daemon (External or Separate Component)]
        PluginDaemon -- Manages --> PluginStorage[Plugin Packages & Manifests Storage]
        PluginDaemon -- Handles --> PluginLifecycle[Installation, Updates, Uninstallation]
    end

    subgraph AgentWorkflowIntegration [Agent/Workflow Integration - Using Plugin Tools]
        G[Agent or Workflow Node] -- Needs Tool --> H[ToolManager]
        H -- Discovers Provider --> PluginToolProviderCtrl[core/tools/plugin_tool/provider.PluginToolProviderController]
        PluginToolProviderCtrl -- Gets Tool Info from Plugin --> PluginDaemon
        PluginToolProviderCtrl -- Instantiates --> PT[core/tools/plugin_tool/tool.PluginTool]
    end

    subgraph PluginToolExecution [Plugin Tool Execution]
        AgentWorkflow[Agent/Workflow] -- Invokes Tool --> PT
        PT -- Uses PluginToolManager --> PTMgr[core/plugin/impl/tool.PluginToolManager]
        PTMgr -- Sends Invoke Request to --> PluginDaemon
        PluginDaemon -- Executes Tool Logic in Plugin --> PluginRuntime[(Plugin's Runtime Environment)]
        PluginRuntime -- (Optional) Interacts with --> ExternalService[Plugin's External Service/API]
        PluginRuntime -- Returns Result --> PluginDaemon
        PluginDaemon -- Streams/Returns Result --> PTMgr
        PTMgr -- Returns Result --> PT
        PT -- Returns Formatted Output --> AgentWorkflow
    end
    
    subgraph CoreDB [Core Application Database]
        PluginService -- Stores Minimal Metadata (e.g., for API-Based Ext) --> DifyDB[(Dify Database - api_based_extensions, etc.)]
        ToolManager -- Reads Provider Credentials --> DifyDBBuiltinProvider[(BuiltinToolProvider Table - for plugin provider credentials)]
    end
```

**Diagram Explanation:**

*   **Plugin Management (Console/API):**
    *   Users manage plugins (install, uninstall, upgrade) through the Dify console.
    *   API requests are handled by `PluginService`.
    *   `PluginService` communicates with a **Plugin Daemon** (likely an external service or a distinct component of the Dify backend) which is responsible for tasks like fetching plugin packages, resolving dependencies, installing plugins, and serving plugin manifests.
    *   The Plugin Daemon manages the storage of plugin packages and their manifests.
*   **Agent/Workflow Integration (Using Plugin Tools):**
    *   When an agent or workflow needs a tool, it requests it from `ToolManager`.
    *   `ToolManager` discovers available tools. If a tool is provided by a plugin, it instantiates `PluginToolProviderController`.
    *   `PluginToolProviderController` might fetch detailed tool metadata from the Plugin Daemon and then creates an instance of `PluginTool`.
*   **Plugin Tool Execution:**
    *   The agent/workflow invokes the `PluginTool`.
    *   `PluginTool` delegates the execution to `PluginToolManager`.
    *   `PluginToolManager` sends an invocation request (including tool name, parameters, and credentials) to the **Plugin Daemon**.
    *   The Plugin Daemon routes this request to the appropriate plugin's runtime environment, where the actual tool logic is executed. This might involve the plugin calling its own external APIs.
    *   The result from the plugin is returned to the Plugin Daemon, then streamed or returned back through `PluginToolManager` and `PluginTool` to the agent/workflow.
*   **Core Application Database:**
    *   While the main plugin code and complex configurations might reside with the Plugin Daemon, Dify's core database stores credentials for plugin providers (within `BuiltinToolProvider` if treated similarly) and configurations for simpler "API-Based Extensions" (which are distinct from the daemon-managed plugins).

## 3. Related Modules

| Path                                               | Role                                                                                         | Major Classes/Functions                                                        |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `api/core/plugin/impl/plugin.py`                   | Manages plugin lifecycle (install, uninstall, upgrade, list) by communicating with a plugin daemon. | `PluginInstaller`                                                              |
| `api/core/plugin/impl/tool.py`                     | Manages interaction with plugin tools, dispatching calls to the plugin daemon.                 | `PluginToolManager`                                                            |
| `api/core/tools/plugin_tool/tool.py`               | Represents a tool provided by a plugin, making it usable by the core tool system.              | `PluginTool`                                                                   |
| `api/core/tools/plugin_tool/provider.py`           | Represents a provider that offers plugin-based tools.                                        | `PluginToolProviderController`                                                 |
| `api/core/plugin/entities/plugin.py`               | Pydantic models for plugin declarations (manifests), installations, and dependencies.        | `PluginDeclaration`, `PluginInstallation`, `PluginEntity`, `GenericProviderID` |
| `api/services/plugin/plugin_service.py`            | High-level service for plugin operations, often wrapping `PluginInstaller` and `PluginToolManager`. | `PluginService`                                                                |
| `api/controllers/console/extension.py`             | API endpoints for managing extensions (includes API-Based Extensions, may touch on plugins).   | `APIBasedExtensionAPI` (Note: Plugin-specific controllers might be elsewhere or abstracted via PluginService) |
| `api/core/tools/tool_manager.py`                   | Discovers and manages all tools, including those from plugins.                               | `ToolManager`, `get_tool_runtime`, `list_builtin_providers`                    |
| `api/models/tools.py`                              | Database models for storing configurations of various tool types (API, Workflow).              | `ApiToolProvider`, `WorkflowToolProvider`, `BuiltinToolProvider` (may store plugin provider credentials) |
| `api/models/api_based_extension.py`                | Database model for simpler, user-configured API-based extensions.                            | `APIBasedExtension`                                                            |

## 4. Data Flow

### a. Plugin Tool Invocation by an Agent

1.  **Input:** An agent (e.g., `FunctionCallAgentRunner` or `CotAgentRunner`) determines it needs to use a tool provided by a plugin (e.g., a specialized data lookup tool from an installed plugin). The agent has the tool's name (e.g., "my_plugin/lookup_data") and the necessary input parameters.
2.  **Tool Retrieval (`ToolManager`):**
    *   The `AgentRunner` requests the tool from `ToolManager.get_agent_tool_runtime()`, specifying `provider_type=ToolProviderType.PLUGIN`, `provider_id="my_organization/my_plugin"`, and `tool_name="lookup_data"`.
    *   `ToolManager`, through `get_builtin_provider()`, identifies that `my_organization/my_plugin` is a plugin. It calls `PluginToolManager.fetch_tool_provider()` to get the `PluginToolProviderEntity` from the plugin daemon.
    *   A `PluginToolProviderController` is instantiated.
    *   `ToolManager` then gets a `PluginTool` instance from this controller. Credentials for the plugin provider (if any are stored in Dify, e.g., in `BuiltinToolProvider` table) are loaded into the `ToolRuntime`.
3.  **Tool Execution (`PluginTool` & `PluginToolManager`):**
    *   The `AgentRunner` calls the `invoke()` method on the `PluginTool` instance (via `ToolEngine.agent_invoke()`).
    *   `PluginTool._invoke()` calls `PluginToolManager.invoke()`.
    *   `PluginToolManager.invoke()` constructs an HTTP request to the plugin daemon's tool invocation endpoint (e.g., `/plugin/{tenant_id}/dispatch/tool/invoke`). This request includes:
        *   Tenant ID, User ID.
        *   Plugin identifier (`X-Plugin-ID` header).
        *   Tool name and parameters.
        *   Credentials (if required by the plugin and configured).
4.  **Plugin Daemon & Plugin Execution:**
    *   The plugin daemon receives the request.
    *   It routes the request to the specific plugin's runtime environment.
    *   The plugin's code for the "lookup_data" tool is executed with the provided parameters and credentials. This might involve the plugin making its own calls to external APIs or services.
    *   The plugin generates a result (text, JSON, files, etc.).
5.  **Response Handling:**
    *   The plugin returns the result to the plugin daemon.
    *   The plugin daemon streams or sends the result back to `PluginToolManager`.
    *   `PluginToolManager` yields `ToolInvokeMessage` objects. If files are involved, it handles chunking (`BLOB_CHUNK`) and final `BLOB` messages.
    *   `ToolEngine` processes these messages, potentially creating `MessageFile` records for binary outputs.
6.  **Output:** The formatted tool output (e.g., a text string summarizing the result) and any associated file IDs are returned to the `AgentRunner` to be used as an "observation."

## 5. Extension Points

*   **Creating New Plugins:**
    1.  Developers create a plugin package, which includes:
        *   A **manifest file** (e.g., `manifest.json` or `manifest.yaml`) defining the plugin's metadata (name, author, version, description, icon), its category (Tool, Model, etc.), resource requirements, and what it provides (tools, models, API endpoints). This maps to `PluginDeclaration`.
        *   The actual code implementing the plugin's logic.
        *   (Optional) A server component if the plugin needs to run its own backend logic, or it can be a purely client-side extension or just a definition for an external API.
    2.  The plugin package is then uploaded to Dify (via UI or API), which interacts with the `PluginService` and `PluginInstaller` to register it with the plugin daemon.
*   **Exposing Tools from Plugins:**
    *   Within the plugin's manifest (`PluginDeclaration.tool`), developers define the `ToolProviderEntity` including the tools offered, their parameters (`ToolParameter`), and how they are invoked.
    *   The plugin's backend implements the logic for these tools.
*   **Exposing Models from Plugins:**
    *   Within the plugin's manifest (`PluginDeclaration.model`), developers define a `ProviderEntity` detailing the AI models the plugin provides, their types, capabilities, and credential requirements.
    *   The plugin's backend implements the logic to interface with these models.
*   **Plugin Marketplace / GitHub Integration:**
    *   The system supports installing plugins from a central marketplace or directly from GitHub repositories, managed by `PluginService` using `source` types like `PluginInstallationSource.Marketplace` or `PluginInstallationSource.Github`.
```
