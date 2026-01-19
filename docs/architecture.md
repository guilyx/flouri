# Architecture of bash.ai

## Overview

`bash.ai` is an AI-powered bash environment enhancement tool designed to provide an intelligent assistant directly within the terminal. It leverages Large Language Models (LLMs) to understand user requests, execute bash commands, and manage a secure environment through allowlists and blacklists. The architecture is modular, allowing for easy extension and integration with various LLM providers via LiteLLM.

## Core Components

The system is composed of several key Python packages:

1. **`bash_ai.config`**:
   - **Purpose**: Manages application settings, environment variables, and persistent configuration (allowlist/blacklist).
   - **Details**: Loads settings from `.env` files and manages `config/commands.json` for dynamic command restrictions.

2. **`bash_ai.agent`**:
   - **Purpose**: Defines the AI agent's behavior, system instructions, and interaction logic with the LLM.
   - **Details**: Constructs the system prompt, including dynamic allowlist/blacklist rules. It uses LiteLLM to communicate with various LLM providers.

3. **`bash_ai.tools`**:
   - **Purpose**: Provides custom Python functions that the AI agent can call to interact with the bash environment and manage command lists.
   - **Details**: Includes `execute_bash`, `set_cwd`, `add_to_allowlist`, `add_to_blacklist`, etc. These tools incorporate pre-execution validation against the allowlist/blacklist.

4. **`bash_ai.runner`**:
   - **Purpose**: Orchestrates the interaction between the user, the AI agent, and the tools.
   - **Details**: Handles sending user prompts to the agent, processing streaming responses, and logging the conversation and tool calls. It acts as the bridge between the UI and the core agent logic.

5. **`bash_ai.ui`**:
   - **Purpose**: Implements the Text User Interface (TUI) and the command-line interface (CLI).
   - **Details**: The TUI provides an interactive chat environment with keyboard shortcuts and displays real-time agent responses and configuration. The CLI offers direct command-line interaction.

6. **`bash_ai.logging`**:
   - **Purpose**: Manages structured logging for sessions, conversations, and tool executions.
   - **Details**: Creates timestamped log files in `~/.config/bash.ai/logs/` and records detailed events for debugging and auditing.

## Data Flow and Interaction

1. **Initialization**:
   - The `bash.ai` application starts (either TUI or CLI).
   - `bash_ai.config` loads settings from `.env` and `config/commands.json`.
   - `bash_ai.logging` initializes a new session log file.

2. **User Input**:
   - The user enters a prompt in the TUI or via the CLI.
   - The `bash_ai.ui` component captures the input.
   - `bash_ai.logging` records the user's message.

3. **Agent Processing**:
   - The `bash_ai.runner` sends the user's prompt to the `bash_ai.agent`.
   - The `bash_ai.agent` constructs a system prompt (including dynamic allowlist/blacklist rules from `bash_ai.config`).
   - The agent uses `LiteLLM` to send the prompt and system instructions to the configured LLM (e.g., Gemini, GPT).

4. **LLM Response and Tool Calling**:
   - The LLM processes the request and generates a response, which might include a decision to call one of the `bash_ai.tools`.
   - If a tool call is suggested (e.g., `execute_bash`), the `bash_ai.runner` intercepts it.

5. **Tool Execution and Validation**:
   - Before executing `execute_bash`, the `bash_ai.tools` module performs a critical security check:
     - It verifies the command against the `blacklist` (always blocked).
     - It checks against the `allowlist`. If the command is not in the allowlist, it triggers a confirmation flow.
   - If confirmation is needed, the system (via `ToolContext.request_confirmation`) prompts the user.
   - **Automated Allowlist Management**: If a command is not in the allowlist and requires confirmation, the agent is instructed to *automatically* call `add_to_allowlist` (after user confirmation for the add operation itself) and then re-execute the original command.
   - `bash_ai.logging` records all tool calls, their parameters, and results.

6. **Output and Display**:
   - The results from tool execution (stdout, stderr, status) or direct LLM text responses are sent back to the `bash_ai.runner`.
   - The `bash_ai.runner` streams or sends the complete response to the `bash_ai.ui`.
   - The `bash_ai.ui` displays the formatted output to the user.
   - `bash_ai.logging` records the agent's response.

## Security Model

The security of `bash.ai` is paramount, especially given its ability to execute arbitrary shell commands.

- **Allowlist/Blacklist**: The core security mechanism.
  - **Blacklist**: Commands explicitly forbidden (e.g., `rm`, `dd`). These are hard-blocked.
  - **Allowlist**: Commands explicitly permitted. If an allowlist is active, only commands on this list can be executed.
  - **Pre-execution Validation**: All commands are validated *before* execution by the `bash_ai.tools` module.
- **User Confirmation**: For commands not in the allowlist (when an allowlist is active), the system requires explicit user confirmation before execution. This is handled by the system, not the agent, to prevent the agent from bypassing security.
- **Automated Allowlist Management**: The agent is instructed to proactively add safe, frequently used commands to the allowlist (after system-level confirmation for the add operation), improving workflow efficiency while maintaining security.
- **Permissions**: The agent operates with the same user permissions as the `bash.ai` application itself. It does not elevate privileges.
- **API Key Security**: API keys are loaded from `.env` files and never committed to version control.

## Extensibility

- **LiteLLM**: Allows easy switching between different LLM providers by changing the `MODEL` environment variable.
- **Modular Tools**: New bash tools can be added by creating Python functions in `bash_ai.tools` and ensuring they are exposed to the agent.
- **Configuration**: The `config/commands.json` provides a flexible way to manage command restrictions.
