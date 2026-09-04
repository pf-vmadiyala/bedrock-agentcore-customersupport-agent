# Customer Support Agent — Amazon Bedrock AgentCore

An e-commerce customer support agent built with the [Strands Agents SDK](https://strandsagents.com) and deployed to
[Amazon Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/). It answers questions about return policies,
product info, and product warranties, can search the web via [Exa AI](https://exa.ai), and remembers facts about
each customer across sessions using AgentCore Memory.

Scaffolded with the [AgentCore CLI](https://github.com/aws/agentcore-cli); this README documents what's actually
in this repo rather than the generic CLI template.

## How it works

```
Browser (Flask UI) ──Cognito JWT──▶ AgentCore Runtime (main.py, Strands agent)
                                          │
                                          ├─ built-in tools: get_return_policy, get_product_info
                                          ├─ AgentCore Gateway ──▶ Lambda: workshop-warranty-check
                                          ├─ Exa AI MCP server (web search)
                                          └─ AgentCore Memory (per-user facts + session summaries)
```

- **`app/CustomerSupport/main.py`** — the agent entrypoint. Builds a Strands `Agent` with the system prompt, local
  tools, and MCP clients, then streams responses back over the AgentCore Runtime HTTP protocol.
- **`app/CustomerSupport/model/`** — model provider setup (Amazon Nova Pro via Bedrock).
- **`app/CustomerSupport/mcp_client/`** — MCP clients for Exa AI web search and the AgentCore Gateway.
- **`app/CustomerSupport/memory/`** — wires up `AgentCoreMemorySessionManager` for per-user semantic memory.
- **`app/CustomerSupport/tool/warranty_schema.json`** — tool schema for the `WarrantyCheck` Gateway target, which
  proxies to a Lambda function (`workshop-warranty-check`) declared in `agentcore/agentcore.json`.
- **`app/CustomerSupport/frontend/`** — a small Flask app that authenticates a demo user against Cognito and serves
  a chat UI (`templates/index.html`) that calls the deployed agent directly from the browser.
- **`agentcore/`** — the declarative AgentCore project: runtime, memory, gateway, and online-eval config
  (`agentcore.json`), deployment target (`aws-targets.json`), and the generated CDK app (`cdk/`) that provisions it
  all. See [AGENTS.md](AGENTS.md) for the full schema reference.

Both the Runtime and the Gateway are protected by a Cognito `CUSTOM_JWT` authorizer — every request carries a
bearer token, and `main.py` extracts the caller's `username` claim from it to scope memory per user.

## Prerequisites

- Node.js 20.x or later
- Python 3.10+ and [uv](https://docs.astral.sh/uv/getting-started/installation/)
- AWS credentials for account `138257835519` / region `us-east-2` (see `agentcore/aws-targets.json`)
- A Cognito user pool matching the `discoveryUrl`/`allowedClients` in `agentcore/agentcore.json`

## Getting started

```bash
agentcore dev      # run the agent locally with hot-reload (localhost:8080)
agentcore invoke --dev "What's the return policy for electronics?"
```

```bash
agentcore deploy   # synthesize CDK and deploy Runtime + Memory + Gateway + online eval to AWS
agentcore status   # check deployment status
```

To run the demo chat UI against a deployed agent:

```bash
cd app/CustomerSupport
python frontend/frontend.py   # http://localhost:8501
```

The UI signs in as the hardcoded workshop user (`frontend/frontend.py`) and reads the Cognito app client ID from
SSM parameter `/app/customersupport/agentcore/web_client_id`.

## Configuration

The `agentcore/` directory is the source of truth — edit the JSON files there, not the generated CDK in
`agentcore/cdk/`. See `agentcore/.llm-context/` for type definitions and `agentcore validate` to check changes.

| Env var | Where | Purpose |
| --- | --- | --- |
| `MEMORY_SHAREDMEMORY_ID` | Runtime | Injected by AgentCore; enables per-user memory retrieval |
| `AGENTCORE_GATEWAY_MY_GATEWAY_SECURE_URL` | Runtime | Injected by AgentCore; Gateway MCP endpoint for `WarrantyCheck` |
| `AWS_REGION` | Runtime / frontend | Deployment region |
| `LOCAL_DEV` | Runtime | Set to `1` to use `agentcore/.env.local` instead of AgentCore Identity |

## Known gaps / suggested improvements

- **Agent instance is cached across all users.** [`main.py`](app/CustomerSupport/main.py:78) keeps the Strands
  `Agent` in a module-level `_agent` singleton and only builds it once. Every request after the first reuses the
  *first* caller's `session_manager` (memory) and Gateway `auth_header` (JWT), regardless of who's actually asking.
  In a multi-user deployment this leaks one user's memory/session into another's conversation and sends stale
  credentials to the Gateway. The agent (or at least its session manager and tool auth) needs to be built per
  request/session instead of cached globally.
- **Two files are dead code**: [`skills/fetcher.py`](app/CustomerSupport/skills/fetcher.py) (S3/git skill fetching)
  and [`model/mantle_compat.py`](app/CustomerSupport/model/mantle_compat.py) (OpenAI-compatible model workaround)
  aren't imported anywhere in `main.py`. Either wire them in or remove them so the codebase reflects what's
  actually running.
- **Hardcoded demo credentials.** [`frontend/frontend.py`](app/CustomerSupport/frontend/frontend.py:33) has a
  plaintext workshop username/password in source. Fine for a disposable workshop account, but don't reuse this
  pattern for anything real — pull from Secrets Manager/SSM instead.
- **Public repo leaks live AWS resource identifiers.** This repo's GitHub remote is public, and
  `agentcore/.cli/deployed-state.json` (intentionally un-gitignored by the AgentCore CLI) plus `agentcore.json`
  commit your real AWS account ID, IAM role ARN, Lambda ARN, and Cognito pool/client IDs. Worth rotating/tearing
  down after the workshop, or moving deploy state to a private repo before reusing this as a long-term project.
- **No tests for the agent code.** The only test in the repo is a boilerplate CDK snapshot test
  (`agentcore/cdk/test/cdk.test.ts`); `get_return_policy`, `get_product_info`, and `extract_user_id` in `main.py`
  have no unit tests.

## Documentation

- [AgentCore CLI](https://github.com/aws/agentcore-cli)
- [AgentCore CDK Constructs](https://github.com/aws/agentcore-l3-cdk-constructs)
- [Amazon Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/)
- [Strands Agents SDK](https://strandsagents.com)
