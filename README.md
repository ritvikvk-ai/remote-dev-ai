# remote-dev-ai

## Description
Remote‑Dev‑AI is an AI‑driven GitHub bot that turns issue requests into pull requests. A GitHub App plus an MCP client/back end listen to repo events, gather context, ask Anthropic Claude to write code, and open a PR with the changes—reducing the time and cost of routine engineering work.

---

## Project Technology Stack

### Integration & Hosting
- **GitHub App + MCP client** (Python) registered as a GitHub App.
- **AWS EC2** runs the MCP client/server for persistent, low‑latency connections.
  - Dockerized **GitHub MCP server** (`ghcr.io/github/github-mcp-server`) provides repo tools.

### Event Handling
- **AWS Lambda** is triggered by GitHub webhooks when a collaborator comments `/remote-dev`.
- Lambda forwards repo/issue metadata to the MCP client’s HTTP endpoint for processing.

### AI Code Generation
- **Anthropic Claude** (via `anthropic`, LangChain, LangGraph) for code synthesis and reasoning.

### Front-end
- **Vercel** (Next.js, generated with v0.dev) for the marketing/installation landing page.

---

## How it works (end-to-end flow)
1. A collaborator comments `/remote-dev` on a GitHub issue.
2. GitHub webhook (via the GitHub App) invokes the **Lambda** handler (`lambda_function.py`).
3. Lambda POSTs `{owner, repo, issue_number, issue_description}` to the MCP client HTTP endpoint (`/process` on the EC2 host).
4. The **MCP client** (`client.py`) connects to the GitHub MCP server (Docker), pulls repo context (README, guidelines, file previews), and runs a LangGraph multi‑agent pipeline (coder → senior → security/performance reviewers → tester) with Claude.
5. The pipeline synthesizes code and prose responses; downstream tooling (not in this repo) would open a pull request with the generated changes.

### Architectural view (textual)
- **GitHub App** → webhook → **Lambda** → HTTP POST → **MCP client (EC2)** → **GitHub MCP server (Docker)** → GitHub API for repo context.
- **LLM layer**: Claude (via LangChain + LangGraph agents).
- **Output**: Code/prose response intended for PR creation and review.

---

## Impact
- **Time savings**: Targets the ~75% of developer time lost to context gathering, boilerplate, and repetitive fixes.
- **Cost efficiency**: At 100 issue calls/day, AI costs ≈\$350/week vs. ≈\$1,200/week in manual dev hours (~70% reduction).
- **Scalability**: GitHub‑native flow works for public/private repos; deploy once, reuse everywhere.
- **Accessibility**: Enterprise‑style automation for small teams without heavy DevOps lift.

---

## Key components in this repo
- `client.py`: Async MCP client that wraps GitHub MCP server tools, builds a LangGraph multi‑agent workflow, and exposes an aiohttp POST endpoint at `/process` to receive Lambda payloads.
- `lambda_function.py`: GitHub webhook handler for issue comments; when `/remote-dev` is posted by a collaborator, it forwards repo/issue metadata to the MCP client endpoint.

---

## Quickstart (self‑hosted preview)
1) **Prerequisites**: Docker (for GitHub MCP server), Python 3.10+, `GITHUB_TOKEN` with repo scope on the target org/user.  
2) **Install deps**: `pip install -r requirements.txt` (or install `anthropic`, `langchain`, `langgraph`, `aiohttp`, `python-dotenv`, `mcp`).  
3) **Run MCP client**: Set env vars and start `client.py`; it should reach `docker run ghcr.io/github/github-mcp-server` to talk to GitHub.  
4) **Expose HTTP**: The aiohttp server listens on `/process` (port used in `lambda_function.py` defaults to `http://54.197.67.80:8080/process`); align ports/host between Lambda and the client.  
5) **Trigger**: In GitHub, comment `/remote-dev` on an issue (App installed); Lambda forwards the payload to your MCP client for processing.

---

## Operational considerations
- **Permissions**: GitHub App needs read/write on issues, PRs, and contents; `GITHUB_TOKEN` must allow repo access for MCP tool calls.
- **Networking**: Lambda must reach the MCP client host/port; configure security groups/firewall accordingly.
- **Context quality**: Better README/guidelines in target repos improve code suggestions (client fetches README and `guidelines.txt` when present).
- **Safety**: Reviewer agents (security/perf/tester) run in the LangGraph loop; add more checks or commit gating as you productize.
- **Observability**: Add logging/metrics around agent verdicts, MCP tool calls, and failure rates to tune prompts and guardrails.
- **Idempotency**: Ensure duplicate `/remote-dev` comments do not double‑process; consider deduping by issue + comment id.

---

## Development challenges
1. **Immature MCP protocol**: GitHub MCP server was newly open‑sourced, requiring protocol exploration and custom wrappers for LangChain tools.  
2. **Context management**: Dynamically fetching README/guidelines and file previews per repo is ongoing to keep AI outputs aligned with house style.

---

## Configuration hints
- `GITHUB_TOKEN`: required by the MCP server to read/write repo data.
- `ANTHROPIC_API_KEY`: for Claude access.
- `TAVILY_API_KEY` (optional): enables Tavily search tool when `use_tavily=True`.
- `POST_API_URL` in `lambda_function.py`: set to the MCP client `/process` endpoint (host:port).
- Port alignment: update the MCP client server port and Lambda `POST_API_URL` together.

---

## Known gaps / next steps
- Harden auth between Lambda and MCP client (shared secret/IP allowlist).
- Add retries/backoff for MCP calls and Claude generations.
- Add unit/integration tests for the Lambda payload contract and MCP client HTTP handler.

---

## Links
- Live landing page: https://v0-remote-dev-ai.vercel.app/
- Demo video: https://www.youtube.com/watch?v=Yo3f0XnsO30
- Frontend source (Vercel/v0.dev): https://github.com/rrgodhorus/hofhacks-remote-dev-ai-home

---

## Team
remote-dev-ai: Yashwanth Alapati, Rajath Reghunath, Manan Prakashkumar Patel, Ritvik Vasantha Kumar
