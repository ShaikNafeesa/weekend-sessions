# Multi-Agent Finance System: MCP + A2A on Amazon Bedrock AgentCore

One MCP server for stock-price tools, one A2A sub-agent, one A2A orchestrator.

```
    ┌──────────────────┐
    │    Finance       │
    │  Orchestrator    │   A2A, port 9000
    └────────┬─────────┘
             │ A2A  (tool: ask_stock_agent)
             ▼
    ┌──────────────────┐
    │   Stock Agent    │   A2A, port 9001 locally (9000 on AgentCore)
    │  + http_request  │
    └────────┬─────────┘
             │ MCP (streamable HTTP)
             ▼
    ┌──────────────────┐
    │   Stock Tools    │   MCP, port 8000, path /mcp
    │ yfinance prices  │
    └──────────────────┘
```

| File | Protocol | Local port | What it does |
|---|---|---|---|
| `stock_mcp_server.py` | MCP | 8000 | `get_stock_price` tool backed by yfinance |
| `stock_a2a_agent.py` | A2A | 9001 | Agent with the MCP tools + `http_request` |
| `orchestrator.py` | A2A | 9000 | Agent that calls the Stock Agent through an `ask_stock_agent` tool |

## Part 1 — Run everything locally

The two agents call Bedrock from your machine, so Terminals 2 and 3 need your AWS profile and a model you can use (see "Before you start" in the section README). Start them in this order, each in its own terminal, from this folder:

```bash
pip install -r requirements.txt

# Terminal 1 — wait for "Uvicorn running on http://0.0.0.0:8000" before going on
python stock_mcp_server.py

# Terminal 2
set -a; source ../../.env; set +a      # AWS_PROFILE, AWS_REGION, and BEDROCK_MODEL_ID if needed
python stock_a2a_agent.py

# Terminal 3
set -a; source ../../.env; set +a
python orchestrator.py
```

- **Wait for the MCP server.** The stock agent connects to it as it starts. The server's first start can take ~30 s; if Terminal 2 starts too early it exits with a connection error (`ConnectError: All connection attempts failed` / `MCPClientInitializationError`) — just start it again.
- **Another app on port 8000?** The MCP server always uses 8000 (AgentCore expects it there). On macOS it can still start next to an app listening on `127.0.0.1:8000`, but `localhost:8000` then reaches the other app, and the stock agent fails with `POST http://localhost:8000/mcp "HTTP/1.1 404 Not Found"` followed by `McpError: Session terminated`. See what's listening with `lsof -nP -iTCP:8000 -sTCP:LISTEN`, and stop the other app. Or, for the local test only, run the MCP server on another port and point the stock agent at it:
  ```bash
  # Terminal 1
  python -c "import stock_mcp_server as s; s.mcp.settings.port = 8010; s.mcp.run(transport='streamable-http')"
  # Terminal 2, before python stock_a2a_agent.py
  export STOCK_MCP_URL=http://localhost:8010/mcp
  ```
- **`Found credentials in shared credentials file` in Terminal 2 or 3** means `AWS_PROFILE` isn't set in that terminal, so calls will use your old default credentials — run the `source` line first.

Talk to the orchestrator (Terminal 4):

```bash
curl -s -X POST http://localhost:9000/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "id": "1", "method": "message/send",
    "params": {"message": {"role": "user",
      "parts": [{"kind": "text", "text": "How is NVDA stock doing today?"}],
      "messageId": "test-1"}}
  }' | jq -r '[.result.artifacts[0].parts[].text] | join("")'
```

Or talk to the Stock Agent directly by changing the port to `9001`. Expect something like "NVIDIA Corporation (NVDA) is currently trading at **$…**, down **…%**". The agents stream their answer, so it arrives split into many `parts`; the `join("")` above glues them back together (`.parts[0].text` alone prints only the first word or letter). If the answer is `null` or empty, run the curl with `| jq .` instead: `"state": "failed"` means the agent hit an error — the reason is in Terminal 2 or 3 (often `ExpiredToken` or the model-access message).

## Part 2 — Deploy all three to AgentCore Runtime

You need the starter toolkit CLI (`pip install bedrock-agentcore-starter-toolkit` — note: AWS also ships a separate Node.js `agentcore` CLI with different commands; this lab uses the Python one) and `jq`:

```bash
pip install bedrock-agentcore-starter-toolkit
export REGION=us-east-1
```

Run all of Part 2 **in one terminal** (the Cognito script exports variables the later steps use), from this lab folder, with your AWS profile set (`set -a; source ../../.env; set +a`). It takes about 25 minutes: each deploy is 2–6 minutes (`configure` also sets up an AgentCore memory for each agent, which `destroy` removes). Every `agentcore` command prints a "Starter Toolkit CLI is no longer supported" recommendation; this lab still works with it — `export AGENTCORE_SUPPRESS_RECOMMENDATION=1` hides the message. Deploys may also print a one-time `Failed to enable observability ... X-Ray Delivery Destination` warning on a fresh account; the deploy continues. If Claude isn't enabled in your account, `export BEDROCK_MODEL_ID=us.amazon.nova-2-lite-v1:0` first — the deploy commands below pass it to the agents.

### Step 1: Cognito user pool for OAuth

From this lab folder (`multi-agent-on-agentcore/`):

```bash
export COGNITO_USERNAME=testuser
export COGNITO_PASSWORD='YourPassword123!'
source ../setup_cognito.sh
```

This exports `POOL_ID`, `CLIENT_ID`, `DISCOVERY_URL` and `BEARER_TOKEN`. The token is valid for 1 hour — run `cognito_refresh_token` in the same shell to get a new one.

Build the authorizer configuration once:

```bash
export AUTH_CONFIG="{\"customJWTAuthorizer\":{\"discoveryUrl\":\"$DISCOVERY_URL\",\"allowedClients\":[\"$CLIENT_ID\"]}}"
```

### Step 2: Deploy the MCP server

```bash
agentcore configure -e stock_mcp_server.py --protocol MCP \
  --requirements-file requirements.txt --authorizer-config "$AUTH_CONFIG" \
  --region $REGION --non-interactive
agentcore deploy --agent stock_mcp_server
```

Copy the agent ARN from the output (or `agentcore status`) and build the MCP URL:

```bash
export MCP_ARN="<stock_mcp_server runtime ARN>"
export STOCK_MCP_URL="https://bedrock-agentcore.$REGION.amazonaws.com/runtimes/$(echo -n $MCP_ARN | jq -sRr '@uri')/invocations?qualifier=DEFAULT"
```

### Step 3: Deploy the Stock Agent

On AgentCore, A2A agents listen on port 9000, so pass `PORT=9000`.

Each deploy copies `$BEARER_TOKEN` into the agent (it uses it to call the next agent), and the token expires 1 hour after you created it. Refresh it right before this step and before Step 4: `cognito_refresh_token`.

```bash
agentcore configure -e stock_a2a_agent.py --protocol A2A \
  --requirements-file requirements.txt --authorizer-config "$AUTH_CONFIG" \
  --region $REGION --non-interactive
agentcore deploy --agent stock_a2a_agent \
  --env PORT=9000 --env STOCK_MCP_URL=$STOCK_MCP_URL --env AGENT_BEARER_TOKEN=$BEARER_TOKEN \
  --env BEDROCK_MODEL_ID=${BEDROCK_MODEL_ID:-global.anthropic.claude-sonnet-4-6}

export STOCK_A2A_ARN="<stock_a2a_agent runtime ARN>"
export STOCK_A2A_URL="https://bedrock-agentcore.$REGION.amazonaws.com/runtimes/$(echo -n $STOCK_A2A_ARN | jq -sRr '@uri')/invocations/"
```

Deploy once more so the agent card advertises the agent's public URL (A2A clients send their messages to the URL in the card):

```bash
agentcore deploy --agent stock_a2a_agent --auto-update-on-conflict \
  --env PORT=9000 --env STOCK_MCP_URL=$STOCK_MCP_URL --env AGENT_BEARER_TOKEN=$BEARER_TOKEN \
  --env BEDROCK_MODEL_ID=${BEDROCK_MODEL_ID:-global.anthropic.claude-sonnet-4-6} \
  --env AGENTCORE_RUNTIME_URL=$STOCK_A2A_URL
```

### Step 4: Deploy the Orchestrator

```bash
agentcore configure -e orchestrator.py --protocol A2A \
  --requirements-file requirements.txt --authorizer-config "$AUTH_CONFIG" \
  --region $REGION --non-interactive
agentcore deploy --agent orchestrator \
  --env STOCK_A2A_URL=$STOCK_A2A_URL --env AGENT_BEARER_TOKEN=$BEARER_TOKEN \
  --env BEDROCK_MODEL_ID=${BEDROCK_MODEL_ID:-global.anthropic.claude-sonnet-4-6}

export ORCH_ARN="<orchestrator runtime ARN>"
```

### Step 5: Test

```bash
curl -s -X POST "https://bedrock-agentcore.$REGION.amazonaws.com/runtimes/$(echo -n $ORCH_ARN | jq -sRr '@uri')/invocations/?qualifier=DEFAULT" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $BEARER_TOKEN" \
  -H "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: $(uuidgen)$(uuidgen)" \
  -d '{
    "jsonrpc": "2.0", "id": "1", "method": "message/send",
    "params": {"message": {"role": "user",
      "parts": [{"kind": "text", "text": "How is NVDA stock doing today?"}],
      "messageId": "test-1"}}
  }' | jq .
```

Expect `"state": "completed"` and the answer split across many `parts` (the agents stream it). To read it as one piece, end the command with `| jq -r '[.result.artifacts[0].parts[].text] | join("")'` instead of `| jq .`. The first call takes 30–45 s (three cold starts in a row).

To confirm the whole chain ran, tail each runtime's log group, `aws logs tail /aws/bedrock-agentcore/runtimes/<agent-id>-DEFAULT --since 10m` (the agent ID is the last part of its ARN): the orchestrator logs a POST to the stock agent's URL, the stock agent logs `POST /`, and the MCP server logs a `CallToolRequest`. A `401` in those logs means the bearer token baked into an agent has expired — run `cognito_refresh_token` and redeploy that agent with `--auto-update-on-conflict`.

### Step 6: Clean up

```bash
agentcore destroy --agent orchestrator --force --delete-ecr-repo
agentcore destroy --agent stock_a2a_agent --force --delete-ecr-repo
agentcore destroy --agent stock_mcp_server --force --delete-ecr-repo
aws cognito-idp delete-user-pool --user-pool-id $POOL_ID --region $REGION
```

`agentcore destroy` removes each runtime with its IAM role, memory, workload identity and uploaded code. Kept, so you can read them later — delete them in the CloudWatch console (or with `aws logs delete-log-group`) when you no longer need them:

- the runtimes' log groups, `/aws/bedrock-agentcore/runtimes/<agent-id>-DEFAULT`
- the memories' log groups, `/aws/vendedlogs/bedrock-agentcore/memory/APPLICATION_LOGS/<memory-id>`
- log delivery settings named after the runtime and memory IDs (free)

The toolkit also leaves a build cache, `.bedrock_agentcore/` (~300 MB), in this folder; it's git-ignored and safe to delete.

## Notes

- **Ports on AgentCore:** HTTP agents use 8080, MCP servers 8000 (`/mcp`), A2A agents 9000. `stock_a2a_agent.py` switches to 9000 automatically inside the AgentCore container.
- **Agent-to-agent auth:** for a classroom demo the deployed agents reuse your Cognito access token (`AGENT_BEARER_TOKEN`), which expires after an hour. In production, give each agent its own machine identity with AgentCore Identity (OAuth2 client-credentials) instead.
- **Model:** set `BEDROCK_MODEL_ID` (e.g. with `--env BEDROCK_MODEL_ID=...`) to change the model.

## Groq edition (no Bedrock models)

`stock_a2a_agent_groq.py` and `orchestrator_groq.py` are the two agents with their model on Groq; `stock_mcp_server.py` has no model and is unchanged. Install `requirements-groq.txt` and set `GROQ_API_KEY` (and optionally `GROQ_MODEL_ID`).

**Local:** the same three terminals, with `python stock_a2a_agent_groq.py` and `python orchestrator_groq.py`. Start the MCP server first and wait for "Uvicorn running" — otherwise the stock agent exits with `MCPClientInitializationError`.

**AgentCore:** the same steps, with the Groq entrypoints and two extra `--env` flags on each agent deploy. Agent names come from the file names, so every `--agent` flag uses `stock_a2a_agent_groq` / `orchestrator_groq`:

```bash
export GROQ_API_KEY=...            # or: set -a; source ../../.env; set +a
export GROQ_MODEL_ID=openai/gpt-oss-120b

# Step 2 — MCP server: unchanged (optionally add --disable-memory --deployment-type container to its configure too)

# Step 3 — Stock Agent
agentcore configure -e stock_a2a_agent_groq.py --protocol A2A \
  --requirements-file requirements-groq.txt --authorizer-config "$AUTH_CONFIG" \
  --region $REGION --non-interactive --disable-memory --deployment-type container
agentcore deploy --agent stock_a2a_agent_groq \
  --env PORT=9000 --env STOCK_MCP_URL=$STOCK_MCP_URL --env AGENT_BEARER_TOKEN=$BEARER_TOKEN \
  --env GROQ_API_KEY="$GROQ_API_KEY" --env GROQ_MODEL_ID=$GROQ_MODEL_ID
# ... then the second deploy with --auto-update-on-conflict and AGENTCORE_RUNTIME_URL, plus the same two GROQ flags

# Step 4 — Orchestrator
agentcore configure -e orchestrator_groq.py --protocol A2A \
  --requirements-file requirements-groq.txt --authorizer-config "$AUTH_CONFIG" \
  --region $REGION --non-interactive --disable-memory --deployment-type container
agentcore deploy --agent orchestrator_groq \
  --env STOCK_A2A_URL=$STOCK_A2A_URL --env AGENT_BEARER_TOKEN=$BEARER_TOKEN \
  --env GROQ_API_KEY="$GROQ_API_KEY" --env GROQ_MODEL_ID=$GROQ_MODEL_ID

# Step 6 — clean up
agentcore destroy --agent orchestrator_groq --force --delete-ecr-repo
agentcore destroy --agent stock_a2a_agent_groq --force --delete-ecr-repo
agentcore destroy --agent stock_mcp_server --force --delete-ecr-repo
aws cognito-idp delete-user-pool --user-pool-id $POOL_ID --region $REGION
```

- The answer streams back in many parts; to print all of it use `jq -r '[.result.artifacts[0].parts[].text] | join("")'` instead of `.parts[0].text`.
- `GROQ_API_KEY` is stored as a runtime environment variable (visible in the console); for production use Secrets Manager or AgentCore Identity.
- `--deployment-type container` builds each image in CodeBuild, so only your source (a few KB) is uploaded; the default direct code deploy uploads a ~100 MB zip per agent, which is painful on a slow connection. `--disable-memory` skips the short-term memory the toolkit would otherwise create for each agent (~2.5 minutes; this lab doesn't use it).
- The Cognito token lasts an hour and is fixed into each agent at deploy time: run `cognito_refresh_token` before every deploy that passes `AGENT_BEARER_TOKEN`, and run Step 5 within the hour.
- Step 6 doesn't remove everything: `agentcore destroy` leaves the `bedrock-agentcore-codebuild-sources-<account>-<region>` S3 bucket, the runtime and CodeBuild log groups, the trace delivery sources/destinations, the `aws/spans` log group and the `TransactionSearchXRayAccess` logs resource policy. Delete them in the console (CloudWatch → Logs, S3) if you don't need them.
- Tested end to end on AgentCore Runtime with `openai/gpt-oss-120b`: orchestrator → Stock Agent → MCP server, one request traced through all three runtimes.
