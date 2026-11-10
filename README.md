# Bedrock AgentCore Customer Support Agent

> **Tech stack:** Python · Amazon Bedrock AgentCore · Strands Agents · Amazon Cognito (JWT) · Cedar policies · LLM evaluation

---

## 🚀 Getting Started

> This project was built as part of the **[Getting Started with Amazon Bedrock AgentCore](https://catalog.workshops.aws/agentcore-getting-started/)** AWS workshop.

### Prerequisites
- Node.js 20+, `uv`, AWS CLI v2, Git
- AWS account with Bedrock model access (Anthropic use-case form submitted)
- AWS region: `us-east-1` or `us-west-2`

### Setup
```bash
npm install -g @aws/agentcore@0.22.0
aws configure
agentcore create --name CustomerSupport --framework Strands --model-provider Bedrock --memory none
cd CustomerSupport
agentcore deploy -y -v
```

Full step-by-step setup, including memory, gateway, JWT auth, policies, and the chat UI, is documented across the labs referenced above.

---

## 🔐 Security Notes

- All credentials, tokens, and `.env.local` files are excluded via `.gitignore`
- The Runtime and Gateway are secured with Cognito JWT authentication
- Sensitive tool inputs (e.g., email addresses in refund reasons) are blocked via semantic policy guardrails
- This repository contains **no live credentials** — reproduce by provisioning your own AWS resources

---

## 📈 What I Learned

Building this reinforced a few things about production agentic systems:
- **Memory is what separates a demo from a product** — without it, every conversation starts from zero.
- **Governance belongs outside the agent** — Cedar policies enforce business rules at the infrastructure boundary, so agent code never needs to "know" about spending limits.
- **Observability shouldn't be an afterthought** — every invocation here is automatically traced via OpenTelemetry, with zero manual instrumentation.
- **Optimization should be evidence-based** — the A/B testing pipeline validates prompt changes with real statistical significance before promotion, not gut feeling.

---

## 👤 Author

**Bikash Kushwaha**
Cloud System Engineer @ Tekbay | BSc Computer Science with AI, Birmingham City University (via Sunway College, Kathmandu)
Full-Stack Developer & Solutions Architect | Focus: Cloud Infrastructure (AWS), AI/ML Pipelines, Backend Development

---

## 📄 License

Released under the [MIT License](LICENSE). Built for learning and portfolio purposes, following AWS's official AgentCore workshop.
