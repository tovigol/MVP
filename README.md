# Transaction Monitor — Server

.NET 8 backend for the Real-Time Financial Monitor assessment.

## Stack

| Concern | Choice |
|---|---|
| API style | Minimal APIs (`Program.cs`) — small surface area, easy to unit/integration test |
| Real-time | SignalR (`TransactionHub`), strongly-typed via `Hub<ITransactionClient>` |
| Validation | FluentValidation |
| Storage | In-memory, `ConcurrentDictionary`-backed (see ADR below for why not SQLite by default) |
| Distributed fan-out | SignalR's Redis backplane (`Microsoft.AspNetCore.SignalR.StackExchangeRedis`), opt-in via config |
| Tests | xUnit, FluentAssertions, NSubstitute, `WebApplicationFactory` for integration tests |

## Project layout

```
src/TransactionMonitor.Api/
  Program.cs                          composition root + all endpoints
  Models/                             Transaction, TransactionStatus, CreateTransactionRequest
  Validation/TransactionValidator.cs  FluentValidation rules
  Storage/                            ITransactionStore + InMemoryTransactionStore
  Hubs/                               TransactionHub, ITransactionClient
  Services/                           ITransactionBroadcaster + SignalR implementation
  Dockerfile
tests/TransactionMonitor.Tests/
  Validation/      rule-level unit tests
  Storage/         store behavior (upsert, ordering, count)
  Concurrency/      many-writer / interleaved-read stress tests
  Services/        broadcaster unit test (mocked IHubContext)
  Integration/     end-to-end HTTP tests via WebApplicationFactory
k8s/
  deployment.yaml  5 replicas, readiness/liveness probes, Redis via Secret
  service.yaml     ClusterIP + session affinity note
  redis-deployment.yaml   demo-only Redis for local clusters
```

## Running locally

```bash
cd src/TransactionMonitor.Api
dotnet run
```

Swagger UI is available at `/swagger` in Development. The React client (see
the companion `client.zip`) expects `POST /api/transactions` and the hub at
`/hubs/transactions`, proxied from `http://localhost:5173`.

## Running tests

```bash
cd tests/TransactionMonitor.Tests
dotnet test
```

## API

| Method | Route | Purpose |
|---|---|---|
| `POST` | `/api/transactions` | Ingests a transaction, validates it, stores it, broadcasts it |
| `GET` | `/api/transactions?take=100` | Latest transactions on *this* pod (see ADR) |
| `GET` | `/api/transactions/{id}` | Single transaction lookup |
| `GET` | `/healthz` | Liveness/readiness probe target |
| WS | `/hubs/transactions` | SignalR hub; clients receive `TransactionReceived(transaction)` |

---

## ADR: synchronizing state across replicas

**Context.** The assessment's stated scenario: deploy 5 pods behind a
Service. A client connected to Pod A's WebSocket won't see a transaction
POSTed to Pod B, because each pod's `InMemoryTransactionStore` and SignalR
connection table are pod-local by default.

**Decision.** Split the problem into two, because they have different
answers:

1. **Real-time fan-out (solved, implemented):** SignalR's Redis backplane.
   Every pod subscribes to the same Redis pub/sub channel. When Pod B's
   ingestion endpoint calls `Clients.All.TransactionReceived(...)`, that
   call is published to Redis; every pod (including Pod A) receives it and
   pushes it down to its own locally-connected clients. This is a few lines
   of configuration (`AddStackExchangeRedis(...)` in `Program.cs`, gated on
   whether `Redis:ConnectionString` is set) and needs no application-level
   pub/sub code — it's the standard, supported way to run SignalR behind
   multiple replicas.

2. **Historical/REST reads (`GET /api/transactions`) — not fully solved by
   the backplane:** the Redis backplane only carries the live broadcast
   message; it doesn't give every pod a shared view of *stored* history.
   Right now, `GET /api/transactions` on Pod A only returns what Pod A has
   personally ingested since it started. For this MVP that's an accepted
   trade-off — the dashboard's primary experience is the live stream, which
   *is* consistent across pods. If the REST history endpoint needs to be
   consistent too, the fix is to swap `InMemoryTransactionStore` for a
   shared backing store (Redis itself as a sorted set / list, or
   Postgres/SQLite-on-shared-volume) behind the same `ITransactionStore`
   interface — the interface was deliberately kept storage-agnostic so this
   is a single-class swap, not a rewrite.

**Why Redis over Kafka/RabbitMQ for the fan-out:** the requirement is
"broadcast the latest event to whoever's currently connected," not
"guarantee every event is durably delivered and replayable." Pub/sub is the
right shape for that; a message broker would add durability and replay
semantics this use case doesn't ask for. If a future requirement is "a
client that reconnects after 10 minutes should catch up on everything it
missed," that's the point to introduce a broker or a persisted event log —
not the broadcast layer.

**Session affinity:** an established WebSocket connection is already pinned
to one pod (it's a single TCP connection), so it needs no extra stickiness
once open. SignalR's `/negotiate` handshake — and any fallback transport
like long polling — does issue separate HTTP requests that must land on the
same pod, so `service.yaml` sets `sessionAffinity: ClientIP` as a safety
net for that handshake and for environments where WebSocket upgrades get
blocked by an intermediary.

**Why in-memory storage instead of SQLite for the MVP:** the spec allows
either. In-memory keeps the concurrency story simple and testable
(`ConcurrentDictionary` gives atomic upserts and snapshot reads with zero
lock code), and since each pod's store is pod-local either way (per the
trade-off above), SQLite-on-local-disk wouldn't actually solve cross-pod
consistency either — it would just add I/O and a migration story for a
problem it doesn't fix. `ITransactionStore` is the seam where a real
implementation (shared SQL store, or SQLite for durability across restarts
of a *single* pod) would plug in without touching the endpoints.

## What's verified vs. not

This project was written and reviewed carefully, but the sandbox used to
build it has no .NET SDK and no access to NuGet — so unlike the React
client (which was `npm install`'d, type-checked, and built end-to-end),
this backend has **not** been compiled or test-run in this environment.
Before relying on it: run `dotnet restore && dotnet build` and
`dotnet test` locally, and skim `Program.cs` for the Redis/CORS wiring
against your actual local ports.
"# MVP" 
##Summary of Changes & Deliverables — Person 2 (AgentGuard MVP)
Stripe MCP Server (stripe_server.py) — Built & Verified on Sandbox

Switched from the official/deprecated Stripe MCP (mcp.stripe.com / @stripe/mcp) to a local Python MCP server over FastMCP + Stripe Python SDK (matching the spec's fallback plan) so existing Gateway and Registry contracts remain unchanged.

Exposes 3 explicit tools: create_payment(amount_cents, recipient, description), list_customers(limit=10), and list_payments(limit=10).

Configured the Sandbox key in .env and verified a real $1.00 Sandbox PaymentIntent (status: succeeded).

CRM MCP Server & Demo Data (crm_server.py, customers.csv) — Built, Hardened & Verified

Built a CSV-backed MCP server with fictional data for "Yossi's Bakery", including customer David Levi (C004) containing the demo prompt injection (export_all + exfiltration to attacker@evil.io).

Security & quality fixes applied: prevented export_all bypass via get_customer (minimum 2-character query and [:3] result cap), safe None handling for missing CSV fields, accurate readOnlyHint: True on export_all, and updated tool docstrings.

Verified tool listing and live lookup of the poisoned David Levi record.

Email MCP Server (email_server.py) — Built & Hardened

Exposes send_email(to, subject, body, cc, bcc) supporting both Resend and Gmail SMTP via .env.

Security & reliability fixes applied: timeout=10 on SMTP, unique make_msgid() for SMTP Message-ID, recipient parsing and validation via getaddresses with newline (\r\n) injection blocking on addresses and subject, logging both successes and failures to sent_emails.jsonl, and destructiveHint: True.

Demo Agent, Prompts & Red Team (agent.py, demo_prompts.txt, redteam.py) — Built & Ready for Integration

agent.py: Strict environment isolation (strips Stripe/Email/Slack keys), Gateway-only connection (http://localhost:8080/mcp), blocked built-in tools (disallowed_tools, setting_sources=[], temporary cwd), MCP_TOOL_TIMEOUT=120000 for Slack approvals, live tool-call printing for the demo, and clean Gateway-offline error handling. An alternative Azure OpenAI + fastmcp.Client version (with zero built-in tools) was also prepared.

demo_prompts.txt: 4 scripted demo scenarios (allowed $80 payment, blocked $5,000 payment, external email Slack approval, and CRM prompt injection).

redteam.py: 10 end-to-end attack/bypass cases (including smurfing, hidden BCC, spoofed domain, comma-concatenated recipients, and CRM @ wildcard bypass) verifying both guard.db events and zero real side effects in Stripe or the email log.

Utilities & Team Coordination (dump_tools.py, .gitignore, requirements.txt)

Added dump_tools.py to export list_tools JSON for Person 3's classify_tools.py, and prepared .gitignore and requirements.txt updates.

Flagged recipient parsing (email.utils.getaddresses()) to Person 1 so the Gateway Policy Engine cannot be bypassed via comma-separated strings or Name <a@b.com> format.

Pending before full integration: Finalizing email provider choice + live send test for email_server.py, running dump_tools.py against the Gateway on port 8080, and running agent.py / redteam.py end-to-end.
##