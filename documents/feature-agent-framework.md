# Agent Framework

## 1. Feature Overview

The Agent Framework provides a flexible system for building and running autonomous agents. These agents can utilize Large Language Models (LLMs), invoke a variety of tools (builtin, API-based, or even other workflows), and follow complex reasoning strategies like Chain-of-Thought (CoT)/ReAct or LLM Function Calling to accomplish tasks. The framework is designed to support different agent behaviors and allow for easy integration and management of tools.

## 2. Architecture Diagram

```mermaid
flowchart TD
    subgraph AgentInvocation [Agent Invocation - App Execution]
        UserInput[User Input / Task] --> AppExecution{Agent Chat App Execution}
        AppExecution -- Gets Agent Config --> AppModelConfig[AppModelConfig (agent_mode)]
        AppExecution -- Instantiates --> AgentRunner[Specific AgentRunner e.g., CotAgentRunner, FunctionCallAgentRunner]
    end

    subgraph AgentCoreLoop [Agent Core Logic & Execution Loop]
        AgentRunner -- Uses --> ModelInstance[LLM Model Instance]
        ModelInstance -- LLM Interaction --> LLMEndpoint[LLM Provider Endpoint]
        
        AgentRunner -- Manages --> PromptEngine[Prompt Engine / Formatter]
        PromptEngine -- Uses --> AgentHistory[Conversation History / Memory]
        PromptEngine -- Uses --> ToolDefinitions[Tool Definitions for Prompt]
        
        AgentRunner -- Identifies Tool Call --> ToolDecision{Tool Call Decision}
        ToolDecision -- Yes --> ToolExecutionPath
        ToolDecision -- No --> FinalResponsePath[Final Response Generation]

        subgraph ToolExecutionPath [Tool Execution]
            AgentRunner -- Gets Tool --> ToolMgr[ToolManager]
            ToolMgr -- Gets Specific Tool --> ToolInstance[BuiltinTool, ApiTool, WorkflowTool, etc.]
            AgentRunner -- Executes Tool --> ToolEng[ToolEngine]
            ToolEng -- Invokes --> ToolInstance
            ToolInstance -- Returns Output --> Observation[Tool Observation]
            Observation --> AgentRunner -- Adds to Scratchpad/History --> PromptEngine
        end
        
        AgentRunner -- Records Steps --> AgentThoughtLog[MessageAgentThought Records]
    end

    subgraph OutputProcessing [Output Processing]
        FinalResponsePath --> OutputParser[AgentOutputParser (e.g., CotOutputParser)]
        OutputParser --> FinalAnswer[Final Answer to User]
        AgentRunner -- Streams Output --> AppExecution
    end

    style AgentInvocation fill:#E6E6FA,stroke:#333,stroke-width:2px
    style AgentCoreLoop fill:#E0FFFF,stroke:#333,stroke-width:2px
    style ToolExecutionPath fill:#FFF0E0,stroke:#333,stroke-width:2px
    style OutputProcessing fill:#F0FFF0,stroke:#333,stroke-width:2px
```

**Diagram Explanation:**

*   **Agent Invocation:** When an agent-enabled application (like an Agent Chat app) receives user input, it retrieves the agent's configuration from `AppModelConfig.agent_mode`. Based on the configured strategy (e.g., "chain-of-thought" or "function-calling"), the appropriate `AgentRunner` (e.g., `CotAgentRunner`, `FunctionCallAgentRunner`) is instantiated.
*   **Agent Core Logic & Execution Loop:**
    *   The `AgentRunner` manages the interaction loop.
    *   **Prompt Engine/Formatter:** Constructs prompts for the LLM, incorporating the original query, conversation history (`Memory`), available tool definitions, and any intermediate thoughts/observations (for CoT).
    *   **LLM Interaction:** The runner sends the prompt to the configured LLM via `ModelInstance`.
    *   **Tool Call Decision:** The LLM's response is analyzed.
        *   For Function Calling agents, the LLM might directly output tool calls.
        *   For CoT agents, an `OutputParser` (e.g., `CotAgentOutputParser`) extracts the next action, which could be a tool call or a final answer.
    *   **Tool Execution:** If a tool needs to be called:
        *   `ToolManager` is used to get a runnable instance of the specified tool.
        *   `ToolEngine.agent_invoke()` handles the actual execution of the tool, passing necessary parameters.
        *   The tool's output (observation) is then fed back into the loop to inform the next LLM call.
    *   **Agent Thought Log:** Each step of the agent's process (thought, action, observation) is logged as a `MessageAgentThought` record.
*   **Output Processing:**
    *   Once the agent determines a final answer (either directly or after tool use), an `OutputParser` might be used to format it.
    *   The `AgentRunner` streams the final answer (and intermediate thoughts/tool calls, depending on the UI implementation) back to the application.

## 3. Related Modules

| Path                                                      | Role                                                                          | Major Classes/Functions                                                                |
| --------------------------------------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `api/core/agent/`                                         | Core agent runners, strategies (CoT, FC), and agent-related Pydantic entities. | `BaseAgentRunner`, `CotAgentRunner`, `FunctionCallAgentRunner`, `AgentEntity`, `AgentToolEntity`, `AgentScratchpadUnit` |
| `api/core/agent/prompt/template.py`                       | Contains prompt templates, especially for ReAct/CoT style agents.             | String constants for prompts.                                                          |
| `api/core/agent/output_parser/cot_output_parser.py`       | Parses LLM output for CoT agents to extract thoughts and actions.             | `CotAgentOutputParser`                                                                 |
| `api/factories/agent_factory.py`                          | Factory for creating agent strategy instances (primarily for plugin agents).  | `get_plugin_agent_strategy`                                                          |
| `api/core/tools/tool_manager.py`                          | Manages discovery, loading, and runtime instantiation of all tool types.      | `ToolManager`, `get_tool_runtime`, `get_agent_tool_runtime`                            |
| `api/core/tools/tool_engine.py`                           | Handles the actual invocation of tools and processing of their responses.     | `ToolEngine`, `agent_invoke`                                                           |
| `api/core/tools/builtin_tool/`                            | Implementations of built-in tools (e.g., Google Search, Wikipedia).           | Specific tool classes like `WebScraperTool`, `TimeTool`                                |
| `api/core/tools/custom_tool/`                             | Logic for user-defined API-based tools.                                       | `ApiToolProviderController`, `ApiTool`                                                 |
| `api/core/tools/workflow_as_tool/`                        | Allows workflows to be used as tools by agents.                               | `WorkflowToolProviderController`, `WorkflowTool`                                       |
| `api/models/model.py`                                     | Defines how agent configurations and thoughts are stored in the database.     | `AppModelConfig` (field `agent_mode`), `MessageAgentThought`                           |
| `api/services/agent_service.py`                           | Service for retrieving agent execution logs.                                  | `AgentService`, `get_agent_logs`                                                       |
| `api/controllers/console/app/agent.py`                    | API endpoint for fetching agent logs.                                         | `AgentLogApi`                                                                          |
| `api/core/app/apps/agent_chat/app_config_manager.py`      | Manages the configuration specific to Agent Chat applications.                | `AgentChatAppConfig`                                                                   |

## 4. Data Flow

### a. Function Calling Agent Cycle

1.  **Input:** User provides a query to an Agent Chat application configured with the "function-calling" strategy.
2.  **Prompt Construction:**
    *   `FunctionCallAgentRunner._organize_prompt_messages()` assembles the prompt history, including system messages, past user/assistant turns, and previous tool calls/responses.
    *   The current user query is added as a `UserPromptMessage`.
    *   Available tools (from `AppModelConfig.agent_mode.tools` and dataset tools) are formatted into `PromptMessageTool` objects and provided to the LLM.
3.  **LLM Invocation (Attempt 1):**
    *   The `ModelInstance.invoke_llm()` method is called with the prompt messages and tool definitions.
    *   The LLM processes the input. It can either:
        *   Respond directly with a text answer.
        *   Respond with one or more "tool_calls" (requests to use specific tools with certain arguments).
4.  **Tool Call Processing (if tool_calls are present):**
    *   `FunctionCallAgentRunner` extracts the tool calls from the LLM's response.
    *   For each tool call:
        *   `ToolManager.get_agent_tool_runtime()` retrieves the runnable tool instance.
        *   `ToolEngine.agent_invoke()` executes the tool with the arguments provided by the LLM. This involves calling the tool's `invoke()` method.
        *   The tool's output (observation) and any generated files are captured.
        *   A `MessageAgentThought` record is created/updated to log the LLM's request for the tool and the tool's input.
    *   The observations from all tool calls are formatted into `ToolPromptMessage` objects.
5.  **LLM Invocation (Attempt 2 - with Tool Responses):**
    *   The `FunctionCallAgentRunner` appends the `AssistantPromptMessage` (containing the initial tool_calls) and the `ToolPromptMessage`s (containing tool observations) to the prompt history.
    *   The LLM is invoked again with this augmented history.
    *   The LLM now uses the tool observations to generate a final textual answer, or potentially more tool calls (if the iteration limit allows).
6.  **Output & Logging:**
    *   The final text response from the LLM is streamed back to the user.
    *   A `MessageAgentThought` record is updated/created to log the final answer or any subsequent thoughts/actions.
    *   All LLM usage (tokens) and tool execution details are logged.

### b. Chain-of-Thought (ReAct) Agent Cycle

1.  **Input:** User provides a query to an Agent Chat application configured with the "chain-of-thought" strategy.
2.  **Prompt Construction:**
    *   `CotAgentRunner._organize_prompt_messages()` assembles the prompt. This includes:
        *   A system prompt defining the ReAct format (Thought, Action, Observation).
        *   Available tool definitions.
        *   Conversation history.
        *   The current user query.
        *   The current `agent_scratchpad` (which is initially empty or contains previous T-A-O steps).
3.  **LLM Invocation:**
    *   The `ModelInstance.invoke_llm()` is called.
    *   The LLM is expected to respond with a "Thought:" followed by an "Action:".
4.  **Output Parsing:**
    *   `CotAgentOutputParser.handle_react_stream_output()` parses the LLM's raw text output stream.
    *   It extracts the "Thought" and the "Action" (which is typically a JSON blob specifying a tool name and input, or "Final Answer").
5.  **Action Execution / Final Answer:**
    *   **If Action is a tool call:**
        *   `ToolManager.get_agent_tool_runtime()` and `ToolEngine.agent_invoke()` are used to execute the tool.
        *   The tool's output becomes the "Observation".
        *   The Thought, Action (tool call), and Observation are added to the `_agent_scratchpad`.
        *   The loop continues from step 2, feeding the updated scratchpad to the LLM.
    *   **If Action is "Final Answer":**
        *   The input to "Final Answer" is taken as the agent's response.
        *   The loop terminates.
6.  **Output & Logging:**
    *   The final answer (or intermediate thoughts/actions if streamed) is sent to the user.
    *   Each T-A-O cycle is logged as a `MessageAgentThought`.

## 5. Extension Points

*   **Adding a New Agent Strategy:**
    1.  Create a new agent runner class inheriting from `api/core/agent/base_agent_runner.py -> BaseAgentRunner`.
    2.  Implement the `run()` method to define the agent's specific logic flow (e.g., how it constructs prompts, processes LLM responses, interacts with tools, and manages its state).
    3.  If the new strategy requires a different way of parsing LLM output, create a corresponding output parser.
    4.  Add a new strategy enum value in `api/core/agent/entities.py -> AgentEntity.Strategy`.
    5.  Update `api/core/app/apps/agent_chat/app_config_manager.py` or similar configuration managers to recognize and allow selection of the new strategy.
    6.  Update the application's execution logic (e.g., in `AgentChatAppRunner` or a factory) to instantiate the new agent runner when its strategy is selected.
*   **Adding New Tools:**
    *   **Built-in Tools:** Develop the tool class under `api/core/tools/builtin_tool/providers/` and register it.
    *   **API Tools (Custom Tools):** Users can define these via the UI, which creates `ApiToolProvider` records. The `ApiToolProviderController` and `ApiTool` classes handle their runtime.
    *   **Workflow-as-Tool:** Existing workflows can be exposed as tools using `WorkflowToolProviderController` and `WorkflowTool`.
    *   **Plugin Tools:** Develop tools as plugins following the plugin system guidelines. `PluginToolManager` and `PluginToolProviderController` handle these.
*   **Customizing Agent Prompts:**
    *   For CoT/ReAct agents, the templates in `api/core/agent/prompt/template.py` can be modified or extended for different languages or more nuanced instructions.
    *   For Function Calling agents, the system prompt provided in the agent configuration (`AppModelConfig.agent_mode.prompt`) is key to guiding LLM behavior.
*   **Modifying Tool Parameter Handling:**
    *   The `ToolParameter` entity in `api/core/tools/entities/tool_entities.py` defines how tool parameters are specified and rendered. This can be extended for new parameter types.
    *   The way agents prepare inputs for tools or how LLMs are prompted to provide arguments for function calls can be adjusted in the respective agent runners.
```
