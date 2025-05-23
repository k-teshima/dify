# Workflow Engine

## 1. Feature Overview

The Workflow Engine provides a robust system for defining, executing, and monitoring complex sequences of tasks within applications. These workflows are structured as directed graphs where nodes represent specific operations (like LLM calls, tool executions, conditional logic, or data transformations) and edges define the flow of data and control. This feature allows users to visually design and manage intricate application logic, moving beyond simple linear processes to create sophisticated, stateful AI interactions.

## 2. Architecture Diagram

```mermaid
flowchart TD
    subgraph WorkflowDefinition [Workflow Definition - Console/API]
        A[UI Workflow Editor] -- JSON Graph & Features --> B{API Request}
        B --> C["/apps/{app_id}/workflows/draft (POST)"]
        C --> D[WorkflowService]
        D -- Stores/Updates --> DraftWorkflow[Workflow Model (Version: draft)]
        DraftWorkflow --> WorkflowTable[(Workflows Table - workflows)]
        
        E[Publish Button UI] --> F{API Request}
        F --> G["/apps/{app_id}/workflows/publish (POST)"]
        G --> D
        D -- Creates from Draft --> PublishedWorkflow[Workflow Model (Version: timestamp)]
        PublishedWorkflow --> WorkflowTable
        AppModel[App Model] -- links to --> PublishedWorkflow
    end

    subgraph WorkflowExecution [Workflow Execution]
        Trigger[Trigger: App Run / Debugger] --> AppGenerateSvc[AppGenerateService]
        AppGenerateSvc --> WorkflowRunSvc[WorkflowRunService]
        WorkflowRunSvc -- Creates --> WR[WorkflowRun Record]
        WR --> WorkflowRunTable[(WorkflowRuns Table)]
        AppGenerateSvc -- Uses --> GraphEngine[core/workflow/graph_engine/GraphEngine]
        
        GraphEngine -- Reads Definition --> PublishedWorkflow
        GraphEngine -- Manages --> VariablePool[core/workflow/entities/VariablePool]
        GraphEngine -- Iterates & Executes Nodes --> NodeExecLoop{Node Execution Loop}
        
        NodeExecLoop --> NodeMapping[core/workflow/nodes/node_mapping.py]
        NodeMapping -- Resolves --> ConcreteNode[core/workflow/nodes/* (e.g., LLMNode, ToolNode)]
        ConcreteNode -- Executes Logic --> ExternalServices[LLMs, Tools, Code Execution, etc.]
        ConcreteNode -- Returns Output --> VariablePool
        ConcreteNode -- Logs Execution --> WorkflowNodeExecRepo[SQLAlchemyWorkflowNodeExecutionRepository]
        WorkflowNodeExecRepo -- Writes --> NodeExecTable[(WorkflowNodeExecutions Table)]
    end

    subgraph LoggingMonitoring [Logging & Monitoring - Console]
        UIRequestsLog[UI Requests for Logs] --> WorkflowRunSvc
        WorkflowRunSvc -- Reads --> WR
        WorkflowRunSvc -- Reads --> NodeExecTable
        UIRequestsLog --> WorkflowService # For viewing workflow definitions
        WorkflowService -- Reads --> WorkflowTable
    end
```

**Diagram Explanation:**

*   **Workflow Definition:**
    *   Users design workflows in the UI, which translates to a JSON graph structure and feature configurations.
    *   `WorkflowService` handles saving this as a "draft" `Workflow` record.
    *   Publishing creates a new versioned `Workflow` record from the draft, and the `App` model is updated to point to this published workflow.
*   **Workflow Execution:**
    *   Triggered by an app run (e.g., a chat message in an advanced chat app) or through the debugger.
    *   `AppGenerateService` likely orchestrates the setup for workflow execution, using `WorkflowRunService` to create a `WorkflowRun` record to track this specific execution.
    *   `GraphEngine` is the core component that takes the workflow definition and executes it.
    *   It uses a `VariablePool` to manage data passed between nodes.
    *   For each node in the graph, `GraphEngine` uses `node_mapping.py` to find the appropriate node implementation class (e.g., `LLMNode`, `ToolNode`).
    *   The concrete node executes its logic (e.g., calling an LLM, executing a tool).
    *   Node outputs are written back to the `VariablePool`.
    *   Execution details of each node are saved via `SQLAlchemyWorkflowNodeExecutionRepository` to the `WorkflowNodeExecutions` table.
*   **Logging & Monitoring:**
    *   The console UI can request workflow run history and detailed node execution logs, which are retrieved by `WorkflowRunService`.
    *   Workflow definitions themselves can be viewed via `WorkflowService`.

## 3. Related Modules

| Path                                                    | Role                                                                      | Major Classes/Functions                                                                 |
| ------------------------------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `api/services/workflow_service.py`                      | Manages CRUD for workflow definitions (draft, published), node configs.   | `WorkflowService`, `sync_draft_workflow`, `publish_workflow`, `run_draft_workflow_node` |
| `api/services/workflow_run_service.py`                  | Manages workflow execution instances and retrieves execution logs.          | `WorkflowRunService`, `get_workflow_run`, `get_workflow_run_node_executions`            |
| `api/models/workflow.py`                                | Database models for workflow definitions, runs, and node executions.      | `Workflow`, `WorkflowRun`, `WorkflowNodeExecution`, `WorkflowDraftVariable`             |
| `api/core/workflow/graph_engine/graph_engine.py`        | Core engine for traversing and executing the workflow graph.              | `GraphEngine`, `run`, `_run_node`                                                       |
| `api/core/workflow/entities/`                           | Pydantic models for workflow graph, nodes, variables, execution state.    | `Graph`, `Node`, `Edge`, `VariablePool`, `GraphRuntimeState`, `NodeRunResult`           |
| `api/core/workflow/nodes/`                              | Base node class and implementations for all node types.                   | `BaseNode`, `LLMNode`, `ToolNode`, `IfElseNode`, `IterationNode`, `AnswerNode`, `EndNode` |
| `api/core/workflow/nodes/node_mapping.py`               | Maps node type strings to their Python implementation classes.            | `NODE_TYPE_CLASSES_MAPPING`                                                             |
| `api/controllers/console/app/workflow.py`               | Console API endpoints for managing and running workflows.                   | `DraftWorkflowApi`, `PublishedWorkflowApi`, `DraftWorkflowRunApi`                         |
| `core/repositories/workflow_node_execution_repository.py` | Repository for saving and retrieving node execution data.                 | `SQLAlchemyWorkflowNodeExecutionRepository`                                             |

## 4. Data Flow

### a. Defining and Publishing a Workflow

1.  **UI Interaction:** A user designs or modifies a workflow in the console UI. This involves adding nodes, configuring them, and connecting them with edges. The UI maintains a JSON representation of this graph and associated features (e.g., conversation opening remarks for a chat workflow).
2.  **Saving Draft:** When the user saves the draft, the UI sends a `POST` request to `/apps/<app_id>/workflows/draft` with the JSON graph and features.
3.  **Service Processing (Draft):**
    *   `api/controllers/console/app/workflow.py -> DraftWorkflowApi.post()` receives the request.
    *   It calls `api/services/workflow_service.py -> WorkflowService.sync_draft_workflow()`.
    *   `WorkflowService` either creates a new `Workflow` record with `version="draft"` or updates the existing one for the app. The `graph` (as JSON string) and `features` (as JSON string) are stored. Environment and conversation variables are also saved.
4.  **Publishing:** When the user publishes the workflow, the UI sends a `POST` request to `/apps/<app_id>/workflows/publish`.
5.  **Service Processing (Publish):**
    *   `api/controllers/console/app/workflow.py -> PublishedWorkflowApi.post()` receives the request.
    *   It calls `WorkflowService.publish_workflow()`.
    *   `WorkflowService` retrieves the current "draft" workflow.
    *   It creates a new `Workflow` record with a new version string (typically a timestamp). The `graph` and `features` are copied from the draft.
    *   The `App` model's `workflow_id` is updated to point to this new published `Workflow` record's ID.
    *   An `app_published_workflow_was_updated` event is dispatched.
6.  **Output:** A new versioned workflow is stored and associated with the application, ready for execution.

### b. Executing a Workflow

1.  **Trigger:** A workflow execution is triggered, for example, by a user sending a message to an Advanced Chat app, or an API call to a Workflow App. User inputs and any relevant context (like conversation ID) are provided.
2.  **Initiation (`AppGenerateService`):**
    *   `AppGenerateService.generate()` (or a similar method) is called.
    *   It retrieves the published `Workflow` associated with the `App`.
    *   It initializes a `VariablePool` with system variables (e.g., user ID, query, files) and conversation variables (if applicable, loaded by `WorkflowEntry` or similar).
3.  **Workflow Run Creation (`WorkflowRunService`):**
    *   `WorkflowRunService` is used to create a `WorkflowRun` record in the database, logging the start of the execution, inputs, and status as "running".
4.  **Graph Execution (`GraphEngine`):**
    *   The `GraphEngine` is instantiated with the workflow graph, initial `VariablePool`, and other context.
    *   `GraphEngine.run()` is called:
        *   It starts execution from the "start" node.
        *   It iterates through nodes based on graph connectivity:
            *   For each node, it instantiates the appropriate `BaseNode` subclass (e.g., `LLMNode`, `ToolNode`) using `NODE_TYPE_CLASSES_MAPPING`.
            *   The node's `run()` method is called. Inputs for the node are resolved from the `VariablePool`.
            *   The node performs its action (e.g., calls an LLM, executes a tool, evaluates a condition).
            *   Node outputs are added back to the `VariablePool`.
            *   The execution status, inputs, outputs, and any errors for the node are logged to `WorkflowNodeExecution` records via `SQLAlchemyWorkflowNodeExecutionRepository`.
            *   `GraphEngine` yields events (`NodeRunStartedEvent`, `NodeRunSucceededEvent`, etc.) to stream progress.
        *   Conditional logic in `IfElseNode` or iteration logic in `IterationNode`/`LoopNode` influences the path of execution.
        *   Parallel branches are executed using a thread pool.
5.  **Completion & Output:**
    *   Execution continues until an "end" node is reached or an unhandled error occurs.
    *   The final outputs from the "end" node (or "answer" node for chat workflows) are collected.
    *   The `WorkflowRun` record is updated with the final status ("succeeded", "failed"), outputs, and total tokens/elapsed time.
6.  **Response:** The result is returned to the caller (e.g., streamed back to the chat UI or returned as an API response).

## 5. Extension Points

*   **Adding a New Custom Node Type:**
    1.  Define a new node type string in `api/core/workflow/nodes/enums.py -> NodeType`.
    2.  Create a new node implementation class in a relevant subdirectory under `api/core/workflow/nodes/` (e.g., `api/core/workflow/nodes/my_custom_node/my_custom_node.py`). This class must inherit from `api/core/workflow/nodes/base/node.py -> BaseNode`.
    3.  Implement the `_run()` method within your custom node class. This method will contain the core logic for what the node does when executed. It should process inputs (from `self.graph_runtime_state.variable_pool` and `self._get_variable_mapping_dict()`) and yield a `RunCompletedEvent` with the node's outputs or errors.
    4.  Define input and output variable schemas using Pydantic models in an `entities.py` file within your node's directory, similar to existing nodes.
    5.  Implement `get_default_config()` in your node class to provide a default configuration structure that will be used by the UI when adding this node to a workflow.
    6.  Register the new node class in `api/core/workflow/nodes/node_mapping.py -> NODE_TYPE_CLASSES_MAPPING` by mapping its `NodeType` enum to your class.
    7.  Update the frontend workflow editor to recognize the new node type, allow users to add and configure it, and define its UI appearance (icon, title, input fields).
*   **Customizing Node Execution Callbacks:**
    *   Implement new callback handlers by inheriting from `api/core/workflow/callbacks/base_workflow_callback.py -> BaseWorkflowCallbackHandler`. These can be used for custom logging, metrics, or external integrations during node execution.
    *   Callbacks can be registered or passed to the `GraphEngine` or `WorkflowEntry` points.
*   **Modifying Variable Resolution:**
    *   The `VariablePool` in `api/core/workflow/entities/variable_pool.py` handles variable storage and retrieval. Its behavior can be extended if different scoping rules or variable types are needed.
*   **Extending Error Handling Strategies:**
    *   New `ErrorStrategy` enums can be added in `api/core/workflow/nodes/enums.py`.
    *   The `GraphEngine`'s error handling logic in `_run_node()` and `_handle_continue_on_error()` would need to be updated to support new strategies.
```
