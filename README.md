# Foundry IQ Knowledge Agent

This project is a working Python command-line client for a Microsoft Foundry prompt agent grounded with a Foundry IQ knowledge base. The sample agent acts as a Contoso outdoor-products expert and answers questions using the product documents in `data/`.

Use this repository as both:

- a learning exercise for understanding Foundry agents, Foundry IQ grounding, conversations, and approval-controlled tool calls; and
- a work instruction for configuring, running, validating, and extending the application.

## Current project status

Implemented now:

- connection to an existing Microsoft Foundry project and prompt agent;
- authentication through `DefaultAzureCredential`;
- a persistent Foundry conversation for each application session;
- multi-turn chat from a terminal;
- client-side conversation history;
- detection and handling of one or more MCP approval requests;
- approval or denial of Foundry IQ tool calls before execution;
- display of the final response and available citations; and
- local product documents for building the Foundry IQ knowledge source.

Not implemented yet:

- browser-based frontend;
- application telemetry and distributed tracing;
- operational dashboards, alerts, and service-level indicators;
- automated evaluation datasets, evaluators, and regression gates;
- automated tests and CI/CD; and
- infrastructure-as-code for the Azure resources.

## Architecture

```text
User
  |
  v
Python CLI (agent_client.py)
  |
  | DefaultAzureCredential
  v
Microsoft Foundry project
  |
  v
Prompt agent
  |
  | approval-controlled knowledge tool call
  v
Foundry IQ knowledge base
  |
  v
Azure AI Search + Contoso product documents
```

The Python application does not upload or index the documents itself. The files in `data/` are used when creating the external knowledge source and knowledge base in Azure. At runtime, the client calls the already-configured Foundry agent by name.

## Project structure

```text
integrate-agent-with-foundry-iq/
|-- README.md
|-- data/
|   |-- contoso-backpacks-guide.md
|   |-- contoso-backpacks-guide.pdf
|   |-- contoso-camping-accessories.md
|   |-- contoso-camping-accessories.pdf
|   |-- contoso-tents-catalog.md
|   |-- contoso-tents-catalog.pdf
|   `-- contoso-products.zip
`-- Python/
    |-- .env.example
    |-- .env                 # local configuration; do not commit
    |-- .venv/               # existing local virtual environment
    |-- agent_client.py
    `-- requirements.txt
```

## Learning objectives

After working through this project, you should be able to:

1. Explain the difference between the client application, prompt agent, and Foundry IQ knowledge base.
2. Configure a prompt agent to ground answers in enterprise documents.
3. Connect to a Foundry project with the Python SDK.
4. Create and reuse a conversation for multi-turn context.
5. Inspect, approve, or deny MCP tool calls before they execute.
6. Validate grounded answers with product-specific test questions.
7. Plan production improvements using tracing, monitoring, and repeatable evaluations.

## Prerequisites

- An Azure subscription with permission to use or create the required resources.
- A Microsoft Foundry project.
- A deployed chat model for the prompt agent.
- An Azure AI Search resource connected through Foundry IQ.
- A prompt agent connected to the Foundry IQ knowledge base.
- Python 3.13 recommended for this lab.
- Azure CLI installed and authenticated.

The current dependencies are defined in `Python/requirements.txt`. `azure-ai-projects` is pinned to `2.3.0`, and `openai` is constrained to a version below 3 for compatibility with that SDK version.

## Part 1: Configure Foundry IQ and the agent

Skip this section if the existing Foundry project, knowledge base, and agent are still available.

### 1. Create or select a Foundry project

In the [Microsoft Foundry portal](https://ai.azure.com), create or select the project that will host the agent. Record its project endpoint; the local application uses it as `PROJECT_ENDPOINT`.

### 2. Prepare the knowledge source

Use the three PDFs in `data/` as the Contoso product knowledge source:

- `contoso-backpacks-guide.pdf`
- `contoso-camping-accessories.pdf`
- `contoso-tents-catalog.pdf`

Upload them to the storage container used by the Foundry IQ knowledge source. The matching Markdown files are retained for readable source inspection and future test-data generation.

### 3. Create the Foundry IQ knowledge base

Connect the Foundry project to Azure AI Search, create a knowledge source for the uploaded documents, and create a knowledge base. Confirm that ingestion completes successfully before attaching it to the agent.

### 4. Configure the prompt agent

Create or select a prompt agent and connect the Foundry IQ knowledge-base tool. A suitable instruction is:

```text
You are a helpful AI assistant for Contoso, specializing in outdoor camping
and hiking products. Always search the knowledge base before answering product
or catalog questions. Give accurate, useful answers, cite the supporting
sources, and clearly say when the knowledge base does not contain the answer.
```

Record the exact agent name; the local application uses it as `AGENT_NAME`.

### 5. Enable approval-controlled tool use

This client is designed to process `mcp_approval_request` items. Configure the agent's Foundry IQ knowledge tool to ask for approval so the user can inspect and allow or deny each lookup. If approval is disabled in the agent configuration, the client still works, but no approval prompt will appear.

## Part 2: Configure the local application

Open a terminal in the `Python` directory:

```powershell
Set-Location .\integrate-agent-with-foundry-iq\Python
```

### 1. Create or activate the virtual environment

An existing `.venv` is currently included locally. Activate it with:

```powershell
.\.venv\Scripts\Activate.ps1
```

If it is missing or must be rebuilt:

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 2. Configure environment variables

Create `.env` from the safe template if necessary:

```powershell
Copy-Item .env.example .env
```

Set both required values:

```dotenv
PROJECT_ENDPOINT=https://<your-foundry-resource>.services.ai.azure.com/api/projects/<your-project>
AGENT_NAME=<your-agent-name>
```

Keep `.env` local because endpoints and future settings may be environment-specific. Commit `.env.example`, not `.env`.

### 3. Authenticate to Azure

```powershell
az login
```

If the account belongs to multiple tenants or subscriptions, authenticate to the correct tenant and select the subscription containing the Foundry project.

The client intentionally excludes environment and managed-identity credentials from `DefaultAzureCredential`, so local execution relies on an available developer credential such as Azure CLI authentication.

## Part 3: Run the application

From the `Python` directory with `.venv` activated:

```powershell
python agent_client.py
```

Expected startup sequence:

1. The `.env` file is loaded and validated.
2. The application authenticates to the Foundry project.
3. The configured agent is retrieved by name.
4. A new conversation is created.
5. The interactive `You:` prompt appears.

Available commands:

- enter a question to send it to the agent;
- enter `history` to display the client-side conversation history;
- enter `quit` to end the session; or
- press `Ctrl+C` to stop the application.

When the agent requests permission to query Foundry IQ, inspect the displayed server, tool name, and arguments. Enter `yes` or `y` to approve; any other response denies that request.

## Part 4: Validate the behavior

Run these checks after changing the agent, its instructions, the knowledge base, dependencies, or client code.

### Smoke test

Ask:

```text
What types of outdoor products does Contoso offer?
```

Confirm that the client connects, an approval request appears when configured, and the answer is grounded in the supplied product documents.

### Retrieval tests

```text
Tell me about the weatherproof features of the tents.
```

```text
What is the difference between the daypacks and expedition backpacks?
```

```text
What camping accessories would you recommend for a weekend hiking trip?
```

Confirm that product names, features, and prices agree with the files in `data/` and that sources are cited when the service returns citation information.

### Conversation-context test

Ask a product recommendation question, then follow it with:

```text
How much do those items cost?
```

Confirm that the second response uses the existing conversation context.

### Approval-denial test

Deny a knowledge lookup and confirm that the client remains usable and the agent does not claim unsupported product details.

### Out-of-scope test

Ask about a product that does not exist in the supplied documents. Confirm that the agent clearly states that the knowledge base does not contain the answer.

## How the client works

`agent_client.py` performs the following sequence:

1. Loads `PROJECT_ENDPOINT` and `AGENT_NAME` from `.env`.
2. Creates `DefaultAzureCredential` and `AIProjectClient` instances.
3. Gets an OpenAI-compatible client from the Foundry project client.
4. Retrieves the prompt agent by name.
5. Creates one conversation for the process lifetime.
6. Adds each user message to that conversation.
7. Requests an agent response with an `agent_reference`.
8. Collects every pending `mcp_approval_request` in the response.
9. Adds approval decisions to the conversation and requests the next response.
10. Repeats the approval loop until no approval requests remain.
11. Prints the final response and stores a local display history.

The Foundry conversation is the authoritative service-side context. `conversation_history` is a separate in-memory list used only by the `history` command and is lost when the process exits.

## Troubleshooting

### Missing configuration

Error:

```text
PROJECT_ENDPOINT and AGENT_NAME must be set in .env file
```

Check that `.env` exists in the `Python` directory and that both values are populated without surrounding placeholder brackets.

### Authentication failure

- Run `az login` again.
- Confirm the active subscription and tenant.
- Confirm that your identity has access to the Foundry project.
- If the account uses restrictive networking, run the client from a network that can reach the project endpoint.

### Agent not found

Confirm that `AGENT_NAME` exactly matches the prompt agent name in the same project identified by `PROJECT_ENDPOINT`.

### No approval prompt

Confirm that approval is enabled on the Foundry IQ knowledge tool used by the agent. The agent may also answer without calling the tool when the question does not require knowledge retrieval.

### Empty or ungrounded answers

- Verify that the knowledge source finished ingesting all three PDFs.
- Confirm that the knowledge base is connected to the agent.
- Test retrieval in the Foundry portal.
- Review the agent instructions and require knowledge lookup for product questions.
- Compare the answer with the source documents in `data/`.

### Dependency errors

Rebuild the environment from `requirements.txt`. Do not remove the `openai<3` constraint unless compatibility with the pinned `azure-ai-projects` version has been verified.

## Security and operational practices

- Never commit `.env`, access keys, tokens, or connection strings.
- Prefer Microsoft Entra ID and least-privilege role assignments over embedded keys.
- Review tool-call arguments before approving them.
- Treat prompts, responses, citations, and traces as potentially sensitive data.
- Define retention and access controls before enabling production telemetry.
- Do not use the sample catalog as evidence that the system is production-ready.
- Delete unused Azure resources to avoid unnecessary cost.

## Future development roadmap

### 1. Frontend UI

Build a browser-based chat interface while keeping the Foundry calls on a backend service.

Planned capabilities:

- streaming assistant responses;
- visible conversation history;
- source and citation panels;
- approval cards that show the requested tool, server, and arguments;
- approve/deny controls for each pending tool call;
- loading, retry, timeout, and error states;
- session reset and conversation selection; and
- accessible, responsive layouts.

Do not expose Azure credentials or call the Foundry project directly from browser code. The backend should own authentication, conversation IDs, approval submission, and response generation.

Suggested delivery order:

1. Refactor the current Foundry interaction into a reusable service module.
2. Add a small authenticated HTTP API around that module.
3. Implement the chat and approval experience in the frontend.
4. Add streaming and citation rendering.
5. Add session persistence and user-level authorization.

### 2. Tracing

Instrument the backend with OpenTelemetry and connect the project or application to Application Insights. Capture a trace across the incoming request, Foundry response call, approval round trip, and final result.

Track at minimum:

- trace and conversation identifiers;
- agent name and deployed version where available;
- model and operation name;
- tool-call name and approval outcome;
- response status and exception type;
- end-to-end and dependency latency; and
- token usage when available and permitted.

Avoid recording secrets or unrestricted prompt/response content. Establish redaction, sampling, retention, and role-based access rules before enabling content capture.

### 3. Observability

Use the trace data to create an operational view in Application Insights or Azure Monitor.

Initial indicators:

- request volume and success rate;
- p50, p95, and p99 response latency;
- Foundry and knowledge-tool dependency latency;
- approval request, approval, and denial rates;
- exceptions grouped by type;
- empty-response rate;
- token consumption and estimated cost where available; and
- retrieval or citation coverage for product questions.

Create alerts for sustained error-rate increases, latency regression, authentication failures, and missing telemetry. Review individual failures by tracing from the application request to the Foundry conversation and dependent tool calls.

### 4. Evaluation

Create a versioned evaluation dataset from the validation questions in this README and additional edge cases derived from `data/`. Each row should include a query and an `expected_behavior` rubric; add ground truth only where a precise answer is useful and maintainable.

Start with a small smoke suite, then add regression and coverage suites. Candidate evaluation dimensions include:

- relevance;
- task adherence;
- intent resolution;
- groundedness and citation quality;
- tool-call accuracy;
- correct handling of approval denial;
- resistance to indirect prompt injection; and
- accuracy for product names, specifications, availability, and prices.

Store evaluation configuration and artifacts under a future `.foundry/` workspace:

```text
.foundry/
|-- agent-metadata.yaml
|-- suites/
|-- datasets/
|-- evaluators/
`-- results/
```

Run the smoke suite before broader tests. After an agent-instruction, model, SDK, or knowledge-base change, run the same regression suite, compare results with the previous version, analyze failure clusters, and require agreed quality thresholds before promotion. Continuous evaluation can later sample production traffic, subject to privacy and retention requirements.

### 5. Delivery and quality automation

After tracing and evaluations are stable:

1. Add unit tests for configuration validation, response extraction, approval loops, and error handling.
2. Add integration tests against a non-production Foundry project.
3. Add formatting, linting, dependency, and secret checks.
4. Run the smoke evaluation suite in CI.
5. Block promotion when critical tests or quality thresholds fail.
6. Provision repeatable development and production environments with infrastructure-as-code.

## Definition of done for future releases

A change is ready to release when:

- local automated tests pass;
- the CLI and frontend smoke tests pass;
- approval and denial paths both work;
- the evaluation suite meets its agreed thresholds;
- traces contain the required correlation fields without leaking secrets;
- dashboards show healthy latency and error rates;
- configuration and rollback steps are documented; and
- a previous stable agent or application version can be restored.

## Reference

- [Microsoft Foundry SDK client libraries](https://learn.microsoft.com/azure/ai-foundry/how-to/develop/sdk-overview)
- [Microsoft Foundry portal](https://ai.azure.com)

