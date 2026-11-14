# Customer Support Agent on Amazon Bedrock AgentCore

> **Tech stack:** Python · Strands Agents · Amazon Bedrock AgentCore (Runtime, Memory, Gateway, Identity, Policy, Evaluations, Harness) · Claude Sonnet 4.6 · Amazon Cognito (JWT) · Cedar · AWS Lambda · AWS CDK · Flask

An e-commerce customer support agent taken from prototype to a governed, observable service on **Amazon Bedrock AgentCore**. The agent answers product and return-policy questions, checks warranties and processes refunds. It remembers customers across sessions and is only allowed to act within business rules enforced **outside** the model.

| Capability | How it's implemented |
|---|---|
| **Serverless agent hosting** | Strands agent on **AgentCore Runtime** (Python, CodeZip build), deployed with the AgentCore CLI and CDK |
| **Long-term memory** | **AgentCore Memory** with a *semantic* strategy (facts per user) and a *summarisation* strategy (summary per session) |
| **Tools via a gateway** | **AgentCore Gateway** exposes two Lambda functions, `check_warranty` and `process_refund`, as MCP tools, with semantic tool search |
| **Authentication** | **Cognito JWT** authorizer on both the Runtime and the Gateway. The gateway reaches tools using an OAuth client-credentials provider |
| **Governance** | **Cedar policies** in an AgentCore Policy Engine: refunds are permitted **only under $100**, and warranty checks only for authenticated users. The gateway currently runs the engine in `LOG_ONLY` mode (decisions are logged for review); switching it to `ENFORCE` blocks violations |
| **Continuous evaluation** | Online **AgentCore Evaluations** on 100% of traffic: goal success rate, correctness and tool-selection accuracy |
| **Prompt A/B testing** | A second runtime reads its system prompt from **configuration bundles** (control vs treatment) before every model call, so prompt variants can be compared without redeploying |
| **No-code agents** | Three **AgentCore Harness** agents defined in JSON: an order researcher (Code Interpreter + Gateway), a report writer with persistent storage, and a container-based dev assistant |
| **Human in the loop** | A test script that asks the order research agent to make a $200 refund, which policy blocks, and checks that it escalates for approval |
| **Chat UI** | A Flask front end that signs in through Cognito and calls the deployed runtime with the user's token |

Built while working through AWS's [Getting Started with Amazon Bedrock AgentCore](https://catalog.workshops.aws/agentcore-getting-started/) workshop.

---

## Architecture

```text
            ┌──────────────┐  Cognito login   ┌──────────────────┐
  User ───▶ │ Flask chat UI │ ───────────────▶ │ Amazon Cognito    │
            └──────┬───────┘   (JWT)          └──────────────────┘
                   │ Bearer token
                   ▼
   ┌────────────────────────────────────────┐        ┌──────────────────────────────┐
   │ AgentCore Runtime: CustomerSupport      │◀──────▶│ AgentCore Memory              │
   │ Strands agent + Claude Sonnet 4.6       │        │  /users/{actor}/facts         │
   │ (JWT authorizer)                        │        │  /summaries/{actor}/{session} │
   └──────────────┬─────────────────────────┘        └──────────────────────────────┘
                  │ MCP
                  ▼
   ┌────────────────────────────────────────┐        ┌──────────────────────────────┐
   │ AgentCore Gateway (JWT, semantic search)│──────▶│ Policy Engine (Cedar)         │
   │                                         │        │  refund amount < $100         │
   └───────┬──────────────────┬──────────────┘        └──────────────────────────────┘
           ▼                  ▼
   Lambda: check_warranty   Lambda: process_refund

   Traces ──▶ AgentCore Observability ──▶ Online Evaluations (GoalSuccess, Correctness, ToolSelection)
```

---

## Example behaviour

| Request | What happens |
|---|---|
| "Is my Smart Watch still under warranty?" | The agent calls `check_warranty` through the Gateway. Policy allows it for an authenticated user |
| "Refund $45 for order ORD-12345" | `process_refund` runs, since the amount is under the $100 limit |
| "Refund $200 for order ORD-55555" | Breaks the $100 Cedar rule. In `ENFORCE` mode the call is denied before it reaches Lambda and the agent escalates; in `LOG_ONLY` mode the violation is recorded |
| A returning customer asks a follow-up | Facts and session summaries are retrieved from Memory (top 3, relevance ≥ 0.3) and added to context |

---

## Repository structure

```text
CustomerSupport/
├── agentcore/
│   ├── agentcore.json          # Declarative project spec: runtimes, memory, gateway, policies,
│   │                           #   evaluations, config bundles, harnesses
│   ├── aws-targets.json        # Deployment account and region
│   └── cdk/                    # CDK app that deploys the spec (TypeScript, with Jest test)
└── app/
    ├── CustomerSupport/        # Main runtime (JWT-protected)
    │   ├── main.py             # Agent entrypoint (Strands + Claude Sonnet 4.6)
    │   ├── memory/session.py   # AgentCore Memory session manager and retrieval config
    │   ├── mcp_client/         # Gateway MCP client
    │   ├── tool/               # warranty_schema.json, refund_schema.json
    │   └── frontend/           # Flask + Cognito chat UI
    ├── CustomerSupportAB/      # Second runtime used for prompt A/B tests
    ├── OrderResearchAgent/     # Harness agent + human-in-the-loop test (test_hitl.py)
    ├── PersistentReportAgent/  # Harness agent writing reports to persistent session storage
    └── ContainerAgent/         # Harness agent running in a custom container image
```

The whole deployment is described in one file, `agentcore/agentcore.json`. Changing the infrastructure means editing that spec and running `agentcore deploy`.

---

## Getting started

**Prerequisites:** Node.js 20+, Python 3.10+ with [`uv`](https://docs.astral.sh/uv/), AWS CLI v2, and an AWS account with Bedrock access to Anthropic models in `us-east-1` or `us-west-2`.

```bash
npm install -g @aws/agentcore@0.22.0
aws configure

git clone https://github.com/bikash9cmd/bedrock-agentcore-support-agent.git
cd bedrock-agentcore-support-agent/CustomerSupport
```

Before deploying, edit `agentcore/agentcore.json` and `agentcore/aws-targets.json`: set your own account, Cognito user pool and Lambda ARNs. The values in this repo point at the workshop account.

```bash
agentcore dev                 # run the agent locally with hot reload
agentcore deploy -y -v        # deploy everything in agentcore.json via CDK
agentcore invoke "Is order ORD-12345 eligible for a refund?"
agentcore traces              # inspect traces; evaluation scores appear in the AgentCore console
```

To run the human-in-the-loop test against a deployed harness:

```bash
export HARNESS_ARN=<order-research-harness-arn>
python3 app/OrderResearchAgent/test_hitl.py
```

---

## Security notes

- Credentials, tokens and `.env.local` files are excluded via `.gitignore`. The repository contains **no secrets**.
- Both the Runtime and the Gateway require a **Cognito-issued JWT**. The gateway uses its own OAuth client to reach tools.
- Business rules (refund limits, who may call which tool) live in **Cedar policies at the gateway**, not in the prompt, so a jailbroken prompt cannot bypass them once the engine is in `ENFORCE` mode.
- To reproduce, provision your own AWS resources. The IDs in the config belong to a workshop account.

---

## What I learned

- **Memory is what separates a demo from a product.** Without it, every conversation starts from zero. Splitting long-term *facts* from per-session *summaries* keeps retrieval relevant.
- **Governance belongs outside the agent.** Cedar policies enforce spending limits at the infrastructure boundary, so the agent code never needs to know about them, and the model can't talk its way past them.
- **Observability shouldn't be an afterthought.** AgentCore traces every invocation and tool call automatically, which is what makes continuous evaluation possible.
- **Optimisation should be evidence-based.** Serving prompts from configuration bundles lets a new prompt be tested against the baseline on real traffic before it's promoted.

---

## Author

**Bikash Kushwaha** · [GitHub](https://github.com/bikash9cmd)
Cloud System Engineer @ Tekbay · BSc Computer Science with AI, Birmingham City University (via Sunway College, Kathmandu)
Focus: cloud infrastructure (AWS), AI/ML pipelines and backend development.

## License

Educational and portfolio project, built following AWS's official AgentCore workshop.
