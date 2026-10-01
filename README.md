🤖 Agentic AI Coding Assistant


A transparent, tool-using coding agent that converts natural-language requirements into generated, validated, executable Python workflows.

✨ Project summary

Agentic_AI_Coding_Assistant.ipynb is an end-to-end reference implementation of an agentic coding workflow. It is designed to do more than return a block of generated code: it maintains context, selects tools, observes execution results, records failures, and iterates toward a useful outcome.

The notebook demonstrates:

•
🧠 LLM-backed code generation

•
🗂️ Short-term session memory and long-term coding preferences

•
🔍 AST-based Python syntax validation and lightweight style checks

•
⚙️ Captured code execution with an isolated in-process namespace

•
🔁 A bounded ReAct-style reasoning/action/observation loop

•
📦 Structured results for generated code, lint findings, execution output, and errors

•
📊 Performance reporting and error analysis

•
🔐 Security-behavior demonstrations and an explicit production security roadmap

The core design principle is inspectable agent behavior. Reasoning, actions, observations, memory updates, and tool results are represented as explicit program state instead of being hidden behind a single chat response.


Implementation note: This is a prototype/reference architecture. It provides namespace isolation, output capture, and exception handling, but Python exec() is not an OS-level security sandbox. See Security model and limitations.




🎯 The problem this project solves

A chat-only coding workflow often leaves the developer with a repetitive manual loop:

1.
Translate a requirement into a prompt.

2.
Copy generated code into a local environment.

3.
Run the code separately.

4.
Interpret failures.

5.
Paste the failure context back into the model.

6.
Repeat without a durable execution history.

This project brings those steps into one explicit engineering loop:

Plain Text


Natural-language request
        ↓
Context retrieval from memory
        ↓
LLM-generated thought and action plan
        ↓
Tool execution: generate → lint → execute
        ↓
Structured observation and error capture
        ↓
Memory update and bounded iteration
        ↓
Actionable result



🏠 Why use a controllable coding agent instead of only a hosted chat app?

Gemini, Claude, ChatGPT, and similar tools are powerful general-purpose assistants. This project is not intended to claim that a prototype automatically replaces them. Its value is different: it gives the developer ownership of the engineering loop.

Dimension
Generic chat workflow
This project’s approach
🎛️ Control
Provider-defined product workflow
You own the prompts, tools, policies, stop conditions, and routing
🔎 Transparency
Conversation context can be difficult to operationalize
Thoughts, actions, observations, and iteration counts are recorded as state
🧩 Integration
Often requires manual copy/paste or platform-specific extensions
Tools are ordinary Python classes that can be replaced or expanded
🔁 Reproducibility
Results may depend on transient conversation context
Artifacts, errors, execution results, and preferences are explicitly tracked
🛡️ Data boundary
Depends on provider settings and product workflow
The control plane runs in your environment; only configured LLM requests leave it
🔌 Model flexibility
Tied to a particular application experience
Uses an OpenAI-compatible endpoint that can be swapped for another provider or local model server
🧪 Engineering feedback
Code generation may be the end of the workflow
Linting, execution, output capture, and error analysis are first-class stages
💰 Cost control
Controlled by a third-party product or plan
You choose the model, token budget, retry policy, and iteration limit
🚀 Extensibility
Bound by available integrations
Add repository search, tests, database tools, CI checks, or approval gates as Python components




This architecture is especially useful when you need:

•
A coding workflow that can be inspected and modified by your team

•
A consistent tool contract across multiple LLM providers

•
Local control over prompts, memory, telemetry, and execution policies

•
A foundation for private enterprise or research workflows

•
Repeatable experiments in tool use, planning, and agent evaluation


In the current notebook, the orchestration layer is local, but the default model call uses Groq’s API. To make the system fully local, point the same client contract at a self-hosted OpenAI-compatible model server.




🧱 System-level functionality

The agent is organized as a small control plane with focused subsystems:

System capability
Responsibility
Why it matters
🧠 Reasoning engine
Interprets the request and proposes the next step
Converts a vague goal into an actionable plan
🧭 Action planner
Selects generate_code, lint_code, execute_code, or complete
Creates a predictable tool vocabulary
🧰 Tool registry
Maps planned actions to Python tool objects
Allows capabilities to be added without rewriting the agent
🗂️ Memory layer
Stores task context, artifacts, errors, preferences, and results
Prevents every interaction from starting from zero
🧪 Validation layer
Parses Python with ast and applies style checks
Catches basic defects before execution or handoff
⚙️ Execution layer
Runs supported Python snippets and captures results
Converts generated code into observable evidence
📤 Output boundary
Separates stdout from stderr
Keeps expected data distinct from diagnostics
📊 Telemetry layer
Measures execution time, success rate, errors, and context size
Creates a starting point for evaluation and optimization
🔐 Safety layer
Uses bounded iterations, error handling, namespace separation, and explicit security tests
Reduces uncontrolled behavior and makes limitations visible
🔌 LLM adapter
Encapsulates provider-specific HTTP communication
Keeps the agent architecture independent from one model vendor




🔄 End-to-end request lifecycle

1.
Receive: the agent accepts a natural-language coding request.

2.
Remember: it summarizes relevant session state and user preferences.

3.
Reason: the LLM proposes what should happen next.

4.
Plan: the LLM selects a constrained action using a JSON-like contract.

5.
Act: the selected tool generates, lints, or executes code.

6.
Observe: the tool returns structured success, output, errors, and metadata.

7.
Update: memory records the artifact or failure.

8.
Iterate: the agent continues until completion or the iteration limit.

9.
Report: the system returns a final result and reasoning trace summary.




🧭 Agent architecture

mermaid

Source



1. GroqLLM — provider adapter 🔌

The GroqLLM class isolates provider-specific communication from the rest of the agent:

•
Calls Groq’s OpenAI-compatible chat-completions endpoint.

•
Uses openai/gpt-oss-20b as the configured model in the notebook.

•
Sends system/user messages, temperature, and a bounded token budget.

•
Extracts assistant content with a fallback for responses exposing a reasoning field.

•
Converts HTTP/API failures into readable error strings.

•
Provides a small replacement boundary for another provider or a local model server.

2. CodingMemory — dual-layer memory 🗂️

The memory system makes state explicit rather than relying only on the model context window.

Short-term session context tracks:

•
Current task

•
Generated code blocks

•
Execution results

•
Errors encountered

•
Variables currently visible to the executor

Long-term coding history stores:

•
Successful patterns

•
Frequent errors

•
Coding-style preferences

•
Preferred libraries

•
Difficulty level

•
Session count and timestamps

get_context_summary() compresses this state into a prompt-ready representation. This provides continuity while keeping the orchestration layer responsible for what is persisted and exposed.

3. CodeGenerator — constrained code generation 🧱

CodeGenerator.generate_code():

•
Accepts a natural-language description and target language.

•
Retrieves the current memory summary.

•
Prompts for clean, readable, commented code with appropriate error handling.

•
Requests code-only output to reduce prose contamination.

•
Removes Markdown code fences if the model returns them.

•
Records the generated artifact, description, language, and timestamp.

•
Returns structured success/error data rather than raw text alone.

4. LinterTool — AST and style feedback 🔍

LinterTool.lint_code() currently supports Python and returns structured findings:

•
Uses ast.parse() for syntax validation.

•
Reports syntax-error line numbers and messages.

•
Flags lines longer than 100 characters.

•
Performs a lightweight function-docstring check.

•
Labels findings as error, warning, or info.

•
Adds lint findings to session memory for later reasoning.

The linter is intentionally dependency-light and explainable. It is not a replacement for Ruff, Pylint, mypy, pyright, or Bandit; those are recommended production extensions.

5. CodeExecutorTool — observable execution ⚙️

CodeExecutorTool.execute_code() demonstrates an execution boundary for Python snippets:

•
Rejects unsupported languages.

•
Uses a dedicated namespace dictionary.

•
Captures stdout and stderr separately.

•
Measures elapsed execution time.

•
Returns output, errors, execution time, visible variables, and success state.

•
Stores execution results and variable-type summaries in memory.

•
Catches syntax/runtime exceptions and returns structured failures.

•
Restores the original output streams in a finally block.

The separation of stdout and stderr allows expected program output to remain machine-readable while diagnostics remain available for debugging.

6. ReActCodingAgent — bounded orchestration 🔁

The agent implements a bounded reasoning/action/observation loop:

1.
Generate a concise thought about the next step.

2.
Plan one action: generate_code, lint_code, execute_code, or complete.

3.
Dispatch the action to the relevant tool.

4.
Record the observation and recent trace.

5.
Continue until completion or max_iterations is reached.

The default maximum is five iterations. This provides a practical control against runaway requests, unpredictable latency, and unnecessary token usage.

The planner requests a JSON object with action, parameters, and reasoning. If parsing fails, it falls back to keyword detection and ultimately defaults to code generation. This is useful for a prototype and is a clear candidate for strict JSON Schema validation in production.

7. AgentAnalyzer — operational visibility 📊

AgentAnalyzer reports:

•
Total code generations

•
Total executions

•
Successful executions

•
Execution success rate

•
Average execution time

•
Error counts grouped by type

•
Number of variables in scope

•
Approximate context size

These metrics form a starting point for regression testing, evaluation datasets, latency analysis, cost measurement, and production observability.

8. SecurityValidator — security behavior checks 🔐

SecurityValidator demonstrates:

•
Malformed syntax handling

•
Runtime exception handling

•
Safe code execution with output capture

•
Namespace isolation behavior

The recorded notebook output shows graceful failure handling, successful output capture, and separation between the executor namespace and the surrounding notebook namespace.




🧰 Libraries and dependencies

Runtime dependencies

Library
Role in the system
requests
Sends HTTP requests to the OpenAI-compatible Groq endpoint
python-dotenv
Loads GROQ_API_KEY from a local .env file
jupyter or notebook
Runs and interacts with the notebook
ipykernel
Provides the Python kernel used by Jupyter




Install them with:

Bash


pip install jupyter ipykernel python-dotenv requests



Python standard library modules

Module
Usage
os
Reads environment variables and configuration
json
Parses planned action responses
datetime
Records timestamps and measures execution duration
typing
Documents structured interfaces with List, Dict, Any, and Optional
ast
Parses Python and detects syntax errors
sys
Redirects and restores standard output/error streams
io.StringIO
Captures in-memory output streams
contextlib
Imported for stream/context-management support; can be used in a future cleanup pass




Why the dependency footprint is intentionally small

The notebook keeps the first implementation lightweight so the architecture is easy to inspect. The agent does not depend on a heavyweight agent framework; the memory model, tools, planner, and execution loop are implemented directly in Python. This makes the system easier to customize, test, and migrate into a larger service.




🚀 Getting started

Prerequisites

•
Python 3.10 or newer recommended

•
Jupyter Notebook or JupyterLab

•
A Groq API key for the default model adapter

1. Create an isolated environment

Bash


python -m venv .venv
source .venv/bin/activate          # Windows: .venv\\Scripts\\activate
python -m pip install --upgrade pip
pip install jupyter ipykernel python-dotenv requests



2. Configure the model provider

Create a .env file in the repository root:

Plain Text


GROQ_API_KEY=your_groq_api_key_here



Never commit credentials. Add this to .gitignore:

Plain Text


.env
.venv/
__pycache__/
.ipynb_checkpoints/



3. Launch the notebook

Bash


jupyter notebook Agentic_AI_Coding_Assistant.ipynb



Run the environment setup and class-definition cells first. Then execute the demonstration cells covering code generation, complex task handling, performance analysis, and security behavior.




🛠️ How to configure the agent in your own project

The notebook is organized as a reference implementation. To use it in a real project, you can begin with the notebook and gradually extract the classes into modules.

Option A: Use directly inside a notebook

After running the setup and initialization cells, the primary entry point is:

Python


request = "Create a function that validates an email address and includes edge-case examples."

result = react_agent.reason_and_act(request)

print(result["success"])
print(result["iterations"])
print(result["result"])



Inspect the generated artifacts and execution history through memory:

Python


latest_code = memory.session_context["generated_code"][-1]
print(latest_code["code"])

print(memory.session_context["execution_results"])
print(memory.session_context["errors_encountered"])



Option B: Extract the notebook into project modules

A maintainable project structure could look like this:

Plain Text


my-project/
├── src/
│   └── agent/
│       ├── __init__.py
│       ├── llm.py          # GroqLLM or another provider adapter
│       ├── memory.py       # CodingMemory
│       ├── tools.py        # CodeGenerator, LinterTool, CodeExecutorTool
│       ├── agent.py        # ReActCodingAgent
│       ├── security.py     # Execution policies and approvals
│       └── telemetry.py    # AgentAnalyzer and traces
├── tests/
├── notebooks/
│   └── Agentic_AI_Coding_Assistant.ipynb
├── .env.example
├── .gitignore
├── pyproject.toml
└── README.md



Then initialize the system from application code:

Python


import os
from dotenv import load_dotenv

from agent.llm import GroqLLM
from agent.memory import CodingMemory
from agent.tools import CodeGenerator, LinterTool, CodeExecutorTool
from agent.agent import ReActCodingAgent

load_dotenv()

llm = GroqLLM(os.environ["GROQ_API_KEY"])
memory = CodingMemory()

code_generator = CodeGenerator(llm, memory)
code_linter = LinterTool(memory)
code_executor = CodeExecutorTool(memory)

tools = {
    "code_generator": code_generator,
    "code_linter": code_linter,
    "code_executor": code_executor,
}

agent = ReActCodingAgent(
    llm=llm,
    memory=memory,
    tools=tools,
    max_iterations=5,
)

result = agent.reason_and_act(
    "Create a CSV analysis function, validate it, and test it with sample data."
)




The import paths above describe the recommended modularized structure. The current repository stores the implementation inside the notebook, so extraction into .py modules is a natural next engineering step.

Project-level configuration checklist

Before integrating this agent into a real application, define:

•
🧠 Model policy: provider, model, temperature, token budget, and fallback model

•
🗂️ Memory policy: what is retained, for how long, and who can access it

•
🧰 Tool policy: which tools are available to the agent and what parameters they accept

•
🔐 Execution policy: whether generated code may access files, network, packages, or processes

•
✅ Approval policy: which operations require human confirmation

•
📊 Observability policy: what is logged, redacted, measured, and traced

•
💸 Budget policy: per-request token, time, and iteration limits

•
🧪 Evaluation policy: how correctness, safety, latency, and tool selection are tested




🧠 Effective problem-solving skills demonstrated

This agent models a repeatable approach to complex coding problems instead of treating every request as a single prompt.

1. Clarify ambiguous requirements

The reasoning layer identifies ambiguity before committing to an implementation. For example, a request to “calculate the palindrome of a number” may mean checking whether a number is a palindrome, reversing its digits, or finding the next palindrome. The agent can surface an interpretation and encode it into the action plan.

2. Decompose large goals

Complex requests are converted into smaller tool-oriented stages:

Plain Text


Understand request
    ↓
Define implementation
    ↓
Generate code
    ↓
Check syntax and style
    ↓
Execute with sample input
    ↓
Inspect output or error
    ↓
Refine or complete



3. Use evidence instead of assumptions

The agent does not need to assume that generated code works. It can use lint findings, execution output, exception data, and timing information as observations for the next decision.

4. Preserve context across steps

Generated artifacts, errors, execution results, current task details, and user preferences remain available through CodingMemory. This supports continuity across a multi-step task.

5. Recover from failure

The workflow treats failures as structured data:

•
API failures become readable provider errors.

•
Invalid planner output triggers fallback action selection.

•
Syntax errors are returned with line information.

•
Runtime errors are captured without terminating the orchestration layer.

•
Unsupported languages return an explicit failure instead of being executed.

6. Keep autonomy bounded

The agent has a maximum iteration count. This is a simple but important production principle: an autonomous system should have explicit limits on time, cost, recursion, and side effects.

7. Make behavior observable

The reasoning trace, action plan, observations, execution duration, success rate, and error categories provide a foundation for debugging and evaluation.

8. Separate responsibilities

The system avoids putting every capability in one prompt or one class:

•
The LLM adapter communicates with the model.

•
Memory manages state.

•
Tools perform focused operations.

•
The ReAct agent orchestrates decisions.

•
The analyzer measures behavior.

•
The security validator exercises boundaries.

This separation makes the system easier to test and evolve.




🧠 LLM strategies used

The notebook uses several practical strategies for agentic coding:

•
🎭 Role separation: generator, reasoning engine, and action planner have distinct system prompts.

•
🌡️ Low temperature: 0.1 encourages more repeatable coding and planning behavior.

•
🗂️ Context retrieval: memory summaries and recent trace steps are injected instead of replaying unlimited history.

•
🎯 Constrained action vocabulary: the planner selects from a small set of known actions.

•
📐 Structured action output: the planner is asked for action, parameters, and reasoning.

•
⏱️ Bounded autonomy: the loop stops after a fixed number of iterations.

•
👀 Observation-driven refinement: tool results are returned to the agent rather than assuming generation succeeded.

•
🧯 Graceful degradation: malformed planner output, API errors, and execution errors become recoverable results.

•
🧩 Separation of concerns: model communication, memory, tools, orchestration, and analysis are independent components.




🔐 Security model and limitations

What the notebook does today

•
Stores the API key in environment variables rather than hard-coding it.

•
Avoids placing the secret in prompts or generated artifacts.

•
Uses a dedicated executor namespace instead of the notebook’s global namespace.

•
Captures exceptions rather than allowing failed snippets to crash orchestration.

•
Separates standard output from diagnostics.

•
Limits agent iterations to reduce runaway requests.

•
Demonstrates syntax-error, runtime-error, and namespace-isolation handling.

Important limitation

Python exec() with a separate dictionary is not a secure sandbox against malicious code. Generated code may still be able to access imports, files, networks, processes, or excessive system resources. Namespace separation is useful for state hygiene, not a complete security boundary.

Production defense-in-depth roadmap

For untrusted or model-generated code, use:

•
📦 Disposable containers or microVMs

•
👤 Non-root execution

•
📁 Read-only filesystems and explicit workspace mounts

•
🌐 Network egress disabled by default

•
⏱️ Wall-clock, CPU, memory, and process-count limits

•
🛡️ Seccomp/AppArmor or equivalent policy controls

•
🧹 Ephemeral workspace cleanup

•
✅ Human approval for file writes, shell commands, network calls, and package installation

•
🧾 Audit logs with secret and personal-data redaction




📊 Demonstrations included in the notebook

Demo 1 — Simple code generation

The agent receives a natural-language request and generates a Python implementation while updating session memory.

Demo 2 — Complex task with tool chaining

The agent receives a CSV/statistics request and demonstrates how a multi-step objective can be routed through generation and execution-oriented tools.

Performance report

The analyzer reports generation counts, execution counts, success rate, average execution time, error categories, variable counts, and approximate context size.

Security behavior tests

The validator checks malformed syntax, runtime failures, safe output, and namespace isolation.




🧪 Recommended production hardening

1.
Replace ad-hoc planner parsing with strict JSON Schema validation.

2.
Add retry/backoff for transient provider errors and rate limits.

3.
Add request budgets, idempotency keys, and cancellation support.

4.
Move execution into a container or microVM with resource and network policies.

5.
Add Ruff, Bandit, mypy/pyright, and project-specific tests.

6.
Add unit tests for every tool and integration tests for the ReAct loop.

7.
Add a persistent, privacy-aware memory store with retention controls.

8.
Redact secrets and sensitive code before external model calls.

9.
Add human approval for destructive or externally visible actions.

10.
Add evaluation suites for correctness, security, latency, cost, and tool-selection accuracy.

11.
Correct the notebook kernel metadata to reflect the intended supported Python version.

12.
Extract the implementation into importable Python modules while keeping the notebook as an executable demonstration.




📁 Repository structure

Current structure

Plain Text


.
├── Agentic_AI_Coding_Assistant.ipynb  # End-to-end implementation and demonstrations
├── README.md                          # Project documentation
└── .env                               # Local secret; never commit



Recommended modular structure

Plain Text


src/agent/
├── llm.py          # Provider/model adapter
├── memory.py       # Session and persistent memory
├── tools.py        # Generator, linter, executor
├── agent.py        # ReAct orchestration
├── security.py     # Policy and execution boundary
└── telemetry.py    # Metrics and traces






💬 Example requests

Plain Text


Create a Python function that calculates the palindrome of a number.



Plain Text


Create a Python function that reads a CSV file and calculates statistics, then test it with sample data.



Plain Text


Create an input validation function, lint it, execute edge-case examples, and report any failures.






🧭 Engineering principles

•
Explicit state beats hidden state. Memory and traces are visible data structures.

•
Tools should be replaceable. Each capability has a focused interface.

•
Every action should produce evidence. Results include status, output, errors, and timing where relevant.

•
Autonomy must be bounded. Iteration limits and approval gates are part of the design.

•
Security claims must match implementation. Namespace isolation is useful, but not equivalent to a container boundary.

•
Local control enables iteration. Prompts, policies, models, telemetry, and integrations can evolve with the repository.

•
Evaluation is a feature. Success rate, errors, latency, and tool-selection quality should be measured continuously.

⚠️ Disclaimer

This repository is an educational and portfolio-quality prototype. Do not execute untrusted or model-generated code on a production host without implementing a real isolation boundary and reviewing the security model for your deployment.

📄 License

Add the license that matches your intended use before publishing this repository. For a permissive open-source portfolio project, MIT is a common choice; consult your organization or project requirements before selecting one.

