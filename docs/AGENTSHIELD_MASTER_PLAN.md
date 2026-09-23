# AgentShield — 24-Hour Full-Stack Master Plan

> **The agent proposes. AgentShield decides.**

This document is the implementation source of truth for the hackathon MVP.

## 1. Executive Summary

AgentShield is an AI-agent security and governance control plane. An AI agent may
propose a structured tool call, but it never has authority to run it. The
AgentShield backend independently verifies agent identity and status, checks tool
permission, calculates deterministic risk, evaluates deterministic policies, and
then **allows**, **requires human approval**, or **blocks** the action. Only an
authorized action reaches a safe mock executor. Each material stage is persisted
and broadcast to the dashboard.

Build a vertical slice—not an enterprise gateway: one Express process, one
Postgres database, a small tool allowlist, and mock tools. It works without an LLM
key using seeded mock agents. Claude, if added, translates natural language into a
validated proposal only; it never makes security decisions.

## 2. Project Scope

| Concept | Meaning in this MVP | Security authority |
|---|---|---|
| User | Authenticated dashboard operator and approver. | May approve a pending request. |
| AI agent | Registered software identity such as `SalesBot`. | May propose, never execute directly. |
| Tool | Server-registered capability such as `CRM.updateBatch`. | Executor dispatches only after authorization. |
| Action | Immutable proposal: agent + tool + JSON input + outcome. | Created by interceptor. |
| Interceptor | Single service/API entry point for actions. | Enforces security pipeline. |
| Risk Engine | Deterministic pure score calculation. | Explains risk; does not execute. |
| Policy Engine | Ordered deterministic rule evaluation. | Chooses decision. |
| Approval system | Server-side pending-decision state machine. | Can release after revalidation. |
| Audit system | Append-only event history. | Supplies evidence and UI feed. |
| Frontend | Existing React visualization/control layer. | Has no authorization authority. |
| Supabase | PostgreSQL persistence and migrations. | Durable state. |
| LLM | Optional intent-to-tool-proposal adapter. | Cannot decide permission/risk/policy/approval. |

**Agent intelligence** decides what it wants to do. **AgentShield security
authority** decides whether that request may execute. The backend—not the agent,
LLM, or frontend—is the security boundary.

**In scope:** agent registration/status, server permissions, allowlisted mock tools,
deterministic risk/policy, approval, audit, realtime dashboard, three polished
demo paths, and a limited mock recovery flow.

**Out of scope:** model training, Kubernetes, microservices, queues/Redis, real
CRM/payment/email integrations, real deletion, a complete MCP gateway, complex
orchestration, multi-tenancy, billing, and universal rollback.

## 3. Final Tech Stack

| Technology | Role / communication | 24-hour fit; mandatory? | Runs | Simpler alternative |
|---|---|---|---|---|
| React + TypeScript | Existing UI; calls REST/socket API. | Preserve prototype; required. | Browser | React JS. |
| Vite | Frontend development/build. | Fast/minimal; required. | Local/Vercel | Existing bundler. |
| Tailwind | Existing visual system. | Keep design intact; required if already used. | Browser | Existing CSS. |
| React Router | Dashboard page routing. | Small/likely present. | Browser | Local page state. |
| TanStack Query | Fetch/cache/mutations/invalidation. | Removes manual async state; recommended. | Browser | `useEffect` state. |
| React Flow | Decision Graph page. | Good visualization; optional. | Browser | Static SVG. |
| Socket.IO client | Live action/approval/audit updates. | Polished, not core. | Browser | Poll/refetch. |
| Zod | Forms/proposal shape validation. | Shared contracts; recommended. | Browser | TypeScript only. |
| Node.js + TypeScript | Single backend runtime. | Familiar and fast; required. | Railway/Render | Node JS. |
| Express | REST/interceptor middleware. | Transparent security boundary; required. | Backend | Fastify. |
| Socket.IO | Persisted-event fanout. | One process/no broker; optional. | Backend | SSE/polling. |
| JWT | Human dashboard session. | Simple MVP auth. | Backend | Fixed demo user fallback. |
| bcrypt | Hash administrator password and agent key. | Standard/small. | Backend | Supabase Auth. |
| Zod | Parse all untrusted bodies/tool inputs. | Essential fail-closed guard; required. | Backend | Manual checks. |
| Helmet/CORS/dotenv | Headers/origin/config. | Minutes to add; required. | Backend | None. |
| Supabase PostgreSQL | Durable relational data. | Hosted Postgres/dashboard; required. | Supabase | SQLite emergency mode. |
| `pg` + SQL migrations | Direct database repositories. | **Recommended**; few moving parts. | Backend | Prisma if already working. |
| Anthropic Claude API | Optional NL-to-proposal adapter. | Adds story, not dependency. | Backend | Mock factory. |
| `MOCK_AGENT_MODE` | Seeded deterministic proposals. | Guarantees demo; required. | Backend | curl payloads. |
| Vercel | Frontend host. | Fast static deployment. | Cloud | Netlify. |
| Railway | Backend/WebSocket host. | Simple env/service deploy. | Cloud | Render. |
| pnpm or npm | Scripts/dependency manager. | Use repository standard. | Dev/CI | — |
| GitHub + Postman | Checkpoints/API verification. | Repeatable collaboration/tests. | Dev | curl only. |

**Database approach:** use `pg` with numbered SQL migrations and compact repository
functions. Do not introduce Prisma unless the team already has it configured: new
generator/migration configuration is a needless 24-hour failure point.

```text
React/Vite/Tailwind ─ REST + Socket.IO ─> Express/TypeScript
                                          ├─ Supabase PostgreSQL via pg
                                          ├─ safe mock tool registry
                                          └─ optional Claude proposal adapter
```

## 4. Architecture

```text
┌───────────────┐ proposal  ┌──────────────────────────────┐
│ Claude / Mock ├──────────>│ AgentShield API (Express)    │
│ AI Agent      │           │ Interceptor                  │
└───────────────┘           └──────────────┬───────────────┘
                                            ▼
                              identity → status → permission
                                            ▼
                               deterministic risk → policy
                              ┌─────────────┼─────────────┐
                              ▼             ▼             ▼
                           ALLOW        APPROVAL       BLOCK
                              │             │             │
                              │        human decision     └─ audit/no execute
                              └─────────────┴──> executor → audit/Postgres
                                                           │
                                             Socket.IO → existing dashboard
```

The executor has no public route. Controllers, the demo runner, and the optional
Claude adapter all call `interceptAction()`, which calls `authorizeToolCall()`.
Only the approved interceptor service can call `executeTool()`.

## 5. Request Lifecycle

For `POST /api/agent-actions`:

1. Verify an `X-Agent-Key` against its stored bcrypt hash (or a signed agent JWT).
   Bind the agent from the credential; never trust `agentId` in the JSON body.
2. Load the agent and append `ACTION_PROPOSED` then `ACTION_INTERCEPTED` audit events.
3. Require `ACTIVE`. `REGISTERED`, `PAUSED`, and `DISABLED` fail closed.
4. Find the tool in the server registry/DB; parse input with that tool's Zod schema.
   Unknown tools are **BLOCKED**, not approved, because an unknown capability must
   not be introduced through a fallback route.
5. Require an enabled `AgentToolPermission` row.
6. Insert the immutable `AgentAction` proposal/input snapshot.
7. Run the pure risk function and persist `RiskAssessment` factors.
8. Run ordered policy evaluation and persist matching keys/evidence with the action.
9. Store final decision/reason, using `BLOCK > REQUIRE_APPROVAL > ALLOW`.
10. For approval: create a pending `ApprovalRequest`, emit event, and stop.
11. For allow (or later approval): reload the agent and re-authorize immediately
    before execution, then dispatch the mock registry function.
12. Persist action result/failure and recovery snapshot when applicable.
13. Append audit events and emit Socket.IO only after successful DB writes.

A paused SalesBot's valid request returns `403` with `decision: "BLOCK"`, records
`ACTION_BLOCKED`, emits `action:blocked`, and **never calls the executor**. A
frontend paused badge is display only; the database check is the control.

## 6. Agent System

An agent is a named, authenticated software principal. Seed these examples:

| Agent | Purpose | Allowed tools |
|---|---|---|
| ResearchBot | Internal research assistant | `KnowledgeBase.read`, `Logs.read` |
| SalesBot | CRM/outreach assistant | `CRM.queryRecords`, `CRM.updateBatch`, `Email.sendBatch` |
| FinanceBot | Finance lookup/refund requester | `Payment.query`, `Payment.refund` |

Store `id`, `name`, `description`, `status`, `apiKeyHash`, `createdAt`, and
`updatedAt`. Do **not** store an agent-level risk value: risk is contextual and
must be stored per `RiskAssessment`; agent summaries are derived from history.

```text
REGISTERED --activate--> ACTIVE --pause--> PAUSED --resume--> ACTIVE
     |                         \--disable--> DISABLED <--disable--+
     +--disable-----------------------------------------------------+
```

- `REGISTERED`: created but cannot propose tool calls until activated.
- `ACTIVE`: eligible for the rest of the server-side pipeline.
- `PAUSED`: temporary administrative stop; every proposal blocks.
- `DISABLED`: revoked; cannot resume in the MVP without explicit admin re-enable.

## 7. Agent Section Implementation

Creation path: **Agents form → `POST /api/agents` → Zod validation → Express
service → DB insert → return agent plus one-time plaintext API key**. Store only
the bcrypt hash. The UI presents permission editing and activation afterward.

Relationship: `Agent 1—* AgentToolPermission *—1 Tool`. SalesBot has enabled rows
for query/update/batch email and no rows for `Payment.refund` or
`CRM.deleteCustomers`; missing or disabled permission blocks before execution.

| Agents page button | API | Real effect |
|---|---|---|
| Register Agent | `POST /api/agents` | Creates registered agent and key. |
| Pause | `POST /api/agents/:id/pause` | Changes DB status, audits/emits. |
| Resume | `POST /api/agents/:id/resume` | Sets active, audits/emits. |
| Disable | `POST /api/agents/:id/disable` | Revokes agent, audits/emits. |
| Edit Permissions | `PUT /api/agents/:id/permissions` | Replaces validated tool IDs. |
| View Actions | `GET /api/agent-actions?agentId=:id` | Opens filtered history. |
| View Details | `GET /api/agents/:id` | Identity, status, permissions, policies, actions/risk/blocks/approvals. |

```http
POST /api/agent-actions
X-Agent-Key: as_live_salesbot_secret
Content-Type: application/json

{"toolName":"Email.sendBatch","input":{"recipientCount":38,"templateId":"q4-followup","external":true}}
```

The details page shows agent identity/status; enabled/denied tools; policy matches;
recent actions; risk history; blocked attempts; and approval history. Every value
comes from an API/database query, never a frontend-only simulation.

## 8. Database Design

Use UUID primary keys, `timestamptz`, `jsonb` for input/evidence, and enums/check
constraints for bounded statuses.

| Model | Key fields, relationships, indexes | Why |
|---|---|---|
| `users` | `id`, `email UNIQUE`, `password_hash`, `role`, timestamps; email index. | Human approvers. |
| `agents` | `id`, `name UNIQUE`, `description`, `status`, `api_key_hash`, timestamps; status index. | Software identity/lifecycle. |
| `tools` | `id`, `name UNIQUE`, `description`, `base_risk`, `input_schema_version`, `reversible`, `enabled`. | Registry metadata mirror. |
| `agent_tool_permissions` | `agent_id FK`, `tool_id FK`, `enabled`; unique `(agent_id, tool_id)`. | Least privilege mapping. |
| `policies` | `id`, `key UNIQUE`, `enabled`, `priority`, `decision`, `conditions jsonb`. | Simple editable deterministic rules. |
| `agent_actions` | `id`, agent/tool FKs, `tool_name`, `input jsonb`, status/decision/reason/result/timestamps; indexes agent+created, status, decision. | Canonical proposal/outcome. |
| `risk_assessments` | `id`, `action_id UNIQUE FK`, score/level/factors/timestamps. | Explainable immutable score. |
| `approval_requests` | `id`, `action_id UNIQUE FK`, status/times/decider FK/comment; status+expiry index. | Approval state machine. |
| `audit_events` | ids/event type/actor ids/decision/risk/reason/metadata/created; indexes time/action/agent/type. | Append-only evidence/UI feed. |
| `recovery_snapshots` | `id`, `action_id UNIQUE FK`, tool, `before_state`, status/restored fields. | Controlled update rollback. |

```text
User ──< ApprovalRequest >── AgentAction ──1 RiskAssessment
                       │             │
Agent ─< AgentToolPermission >─ Tool │──< AuditEvent
Agent ────────────────────────────────┘──1 RecoverySnapshot
Policy ── evaluated against AgentAction (matched keys stored as evidence)
```

Must exist: agents, tools, permissions, actions, risk, approvals, audit. Keep policy
conditions and matched policy evidence as JSON rather than adding more tables. Add
later: organizations, key rotation, policy versions, webhooks, retention, hash chain.

## 9. Authorization

Authentication answers **who is calling?** Authorization answers **may this
credential use this tool with this input now?** Centralize it:

```ts
authorizeToolCall({ agent, toolName, input, permissions, policies }): {
  decision: 'ALLOW' | 'REQUIRE_APPROVAL' | 'BLOCK';
  riskScore: number; riskLevel: RiskLevel; riskFactors: RiskFactor[];
  matchedPolicies: PolicyMatch[]; reason: string;
}
```

It fails closed for inactive agents, unknown/disabled tools, invalid input, and
missing enabled permission; otherwise it combines risk and policies. Every source
(controller, approval release, demo runner, Claude adapter) calls it. This is why
no tool call can bypass the security boundary.

## 10. Risk Engine

```ts
const RISK_WEIGHTS = {
  base: { 'KnowledgeBase.read': 10, 'CRM.queryRecords': 18,
    'CRM.updateBatch': 35, 'Email.sendBatch': 35, 'Payment.refund': 45,
    'CRM.deleteCustomers': 60 },
  scope: { one: 0, batchUnder10: 5, batch10To50: 15, batchOver50: 30 },
  sensitivity: { public: 0, internal: 5, customer: 15, financial: 25 },
  externalImpact: { none: 0, internal: 5, external: 15, financial: 25 },
  irreversibility: { reversible: 0, difficult: 15, destructive: 30 }
} as const;
```

`score = clamp(base + scope + sensitivity + externalImpact + irreversibility,0,100)`.
Levels: LOW 0–29; MEDIUM 30–59; HIGH 60–79; CRITICAL 80–100.

| Tool/input | Calculation | Result |
|---|---|---|
| `KnowledgeBase.read` | 10 + 0 + 0 + 0 + 0 | 10 LOW |
| `CRM.queryRecords` | 18 + 0 + 5 + 0 + 0 | 23 LOW |
| `CRM.updateBatch(12)` | 35 + 15 + 15 + 5 + 0 | 70 HIGH |
| `Email.sendBatch(38)` | 35 + 15 + 5 + 15 + 0 | 70 HIGH |
| `Payment.refund($1200)` | 45 + 5 + 25 + 25 + 15 | 115 → 100 CRITICAL |
| `CRM.deleteCustomers(147)` | 60 + 30 + 15 + 15 + 30 | 150 → 100 CRITICAL |

```json
{"riskScore":100,"riskLevel":"CRITICAL","riskFactors":[
 {"name":"base","points":60},{"name":"scope:batchOver50","points":30},
 {"name":"data:customer","points":15},{"name":"externalImpact","points":15},
 {"name":"irreversibility:destructive","points":30}]}
```

This is explainable because the fixed factors, not an LLM opinion, are persisted.

## 11. Policy Engine

Policies are enabled JSON conditions with numeric priorities. Evaluate all matches;
select highest priority and resolve ties with `BLOCK > REQUIRE_APPROVAL > ALLOW`.

| Key / priority | Condition | Decision |
|---|---|---|
| `paused_agent_guard` / 1000 | agent status is not ACTIVE | BLOCK |
| `bulk_delete_guard` / 900 | delete tool or destructive count > 50 | BLOCK |
| `high_value_refund_guard` / 800 | refund >= $500 | REQUIRE_APPROVAL; block >= $5000 |
| `external_email_review` / 700 | external batch email | REQUIRE_APPROVAL |
| `critical_risk_block` / 200 | score >= 80 | BLOCK |
| `high_risk_review` / 100 | score 60–79 | REQUIRE_APPROVAL |

Email 38 matches both review policies and waits; deletion 147 matches critical and
bulk-delete guards, so blocks. Store matching keys and human-readable reasons.

## 12. Tool System and Optional LLM

The tool registry is server code—not user-configured executable behavior. Include
`CRM.queryRecords`, `CRM.updateBatch`, `CRM.deleteCustomers`, `Email.send`,
`Email.sendBatch`, `Payment.query`, `Payment.refund`, `KnowledgeBase.read`, and
`Logs.read`, each with metadata, a Zod input schema, reversibility flag, and a safe
mock implementation. Mocks return deterministic JSON and do not contact external
systems. An executor exception sets action `FAILED` and appends `ACTION_FAILED`.

No agent may import or call a mock function directly. The registry is dispatchable
only after the interceptor marks the action eligible.

With `MOCK_AGENT_MODE=true`, a factory supplies structured proposals:
ResearchBot/read → allow; SalesBot/email 38 → approval; SalesBot/delete 147 → block.
The project therefore works with no `ANTHROPIC_API_KEY`. If Claude is added later,
it produces only `{toolName,input}` from a user request. Parse it with the exact
same Zod schema, then send it to the same interceptor. Claude never assigns risk,
permissions, policies, approval, or execution outcome.

## 13. Approval System

State machine: `PENDING → APPROVED | DENIED | EXPIRED`; terminal states are
immutable. On approve/deny, transact/lock: load pending and unexpired request,
write the decision/audit, and for approval reload agent then re-run authorization
before execution. If agent is paused/disabled while waiting, block action instead.
Expire lazily on read/decision after 15 minutes; no queue is necessary.

| Endpoint | Effect |
|---|---|
| `GET /api/approvals?status=PENDING` | Approval inbox. |
| `POST /api/approvals/:id/approve` | Authenticated user approves, revalidates, executes. |
| `POST /api/approvals/:id/deny` | Authenticated user denies; it can never execute. |

If server execution fails, keep `APPROVED`, set action `FAILED`, audit error, and
do not automatically retry a possibly consequential operation.

## 14. Audit System

Use one `recordAuditEvent()` helper inside the same DB transaction as transitions.
Record: `ACTION_PROPOSED`, `ACTION_INTERCEPTED`, `RISK_EVALUATED`,
`POLICY_EVALUATED`, `APPROVAL_CREATED`, `APPROVAL_APPROVED`, `APPROVAL_DENIED`,
`ACTION_BLOCKED`, `ACTION_EXECUTING`, `ACTION_EXECUTED`, `ACTION_FAILED`,
`AGENT_PAUSED`, `AGENT_RESUMED`, `RECOVERY_CREATED`, `RECOVERY_RESTORED`.

Every record includes timestamp, type, actor, agent/action/user IDs, risk/decision,
reason, and non-secret JSON evidence. `GET /api/audit?agentId=&actionId=&type=&decision=&q=&from=&to=&page=` powers filtering/search/details; `GET /api/audit/export.csv`
exports matching rows. The Audit Log must be database-backed.

## 15. Realtime

After persistence emit: `action:proposed`, `action:intercepted`,
`action:risk-evaluated`, `action:policy-evaluated`, `approval:created`,
`approval:updated`, `action:blocked`, `action:executing`, `action:executed`,
`action:failed`, `agent:paused`, `agent:resumed`, `audit:created`.

Overview listens to action/approval/audit summaries; Live Actions to action events;
Approvals to approval updates; Agents to agent/action updates; Audit to audit events;
Decision Graph to a selected action trace. Socket handlers invalidate/update TanStack
Query cache; REST remains the source of truth if Socket.IO is disconnected.

## 16. Frontend Pages

| Page | Real data/API | User action; mock boundary |
|---|---|---|
| Overview | `GET /api/overview`, aggregate actions/approvals/audit. | Open filtered operational views; charts may be simple. |
| Live Actions | `GET /api/agent-actions`. | Live event timeline/detail; no fake timers. |
| Approvals | `GET /api/approvals`. | Approve/deny mutation. |
| Decision Graph | `GET /api/agent-actions/:id/trace`. | Render persisted stages; static fallback allowed. |
| Risk Engine | `POST /api/risk/simulate`. | Real deterministic what-if calculation. |
| Policies | `GET/PUT /api/policies`. | Toggle/edit safe fields, not arbitrary code. |
| Agents | agents/detail endpoints. | Register/pause/resume/disable/permissions. |
| Audit Log | audit/list/export. | Filter/search/details/export persisted data. |
| Recovery | snapshots endpoints. | Restore only mock update snapshot. |
| Threat Protection | overview/audit/policy results. | Real blocked evidence; composition may be presentational. |
| Demo | `POST /api/demo/:scenario`. | Runs actual pipeline over mock tools. |

Use `src/lib/api.ts`, `src/lib/socket.ts`, and per-feature query hooks. Preserve
components/layout/styles; replace fake arrays and timeout decisions one page at a
time. React never computes authorization.

## 17. API Specification

Dashboard routes use `Authorization: Bearer <JWT>`; health/login are public; agent
action uses agent credentials. Error envelope:
`{ "error": { "code": "...", "message": "...", "details": [] } }`.
Use 400 validation, 401 authentication, 403 security denial, 404 missing, 409 state,
and generic 500 errors without secrets.

| Group | Method/path | Body/query | Purpose/response |
|---|---|---|---|
| Auth | `POST /api/auth/login` | `{email,password}` | `{token,user}`. |
| Agents | `GET /api/agents`, `GET /api/agents/:id` | `?status=&q=` | Lists/detail. |
| Agents | `POST /api/agents` | `{name,description}` | `{agent,apiKey}` one-time key. |
| Agents | `POST /api/agents/:id/{pause,resume,disable}` | — | Updated agent. |
| Agents | `PUT /api/agents/:id/permissions` | `{toolIds:string[]}` | Updated permission rows. |
| Tools | `GET /api/tools` | `?enabled=true` | Tool metadata. |
| Actions | `POST /api/agent-actions` | agent key + `{toolName,input}` | `{action,authorization,approval?}`. |
| Actions | `GET /api/agent-actions`, `GET /api/agent-actions/:id` | filters | History/detail. |
| Actions | `GET /api/agent-actions/:id/trace` | — | Ordered audit trace. |
| Approvals | `GET /api/approvals` | `?status=&page=` | Inbox. |
| Approvals | `POST /api/approvals/:id/{approve,deny}` | `{comment?}` | Updated request/action. |
| Policies | `GET /api/policies`, `PUT /api/policies/:id` | safe fields | Read/update rules. |
| Risk | `POST /api/risk/simulate` | `{toolName,input}` | Factors/score/level. |
| Audit | `GET /api/audit`, `GET /api/audit/export.csv` | listed filters | Events/CSV. |
| Recovery | `GET /api/recovery`, `POST /api/recovery/:id/restore` | — | Snapshot/restore. |
| Overview | `GET /api/overview` | — | Dashboard aggregate. |
| Demo | `POST /api/demo/:scenario` | `allow|approval|block` | Actual seeded proposal. |
| Health | `GET /api/health` | — | `{status:"ok",database:"ok"}`. |

## 18. Folder Structure

```text
frontend/
  src/{pages,components,features,lib/api.ts,lib/socket.ts,schemas,types}/
backend/
  src/
    server.ts app.ts config/env.ts socket.ts
    routes/{auth,agents,actions,approvals,tools,policies,risk,audit,recovery,demo}.ts
    middleware/{auth,agentAuth,errorHandler}.ts
    services/{interceptor,authorization,risk,policy,approval,executor,audit,recovery}.ts
    tools/{registry,mockTools}.ts db/{pool,migrations,seed,repositories}.ts
    schemas/ types/
  tests/{authorization,risk,policy,actions,approvals}.test.ts
  .env.example
```

Create in this order: backend scripts/env/health, DB migration/seed/pool, registry
and schemas, authorization/interceptor/action route, then approval/execution/audit,
then UI adapters. Avoid generic event buses and unnecessary abstractions.

## 19. Environment Setup

`backend/.env.example`:

```dotenv
NODE_ENV=development
PORT=4000
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/postgres?sslmode=require
JWT_SECRET=replace-with-a-long-random-secret
CLIENT_URL=http://localhost:5173
MOCK_AGENT_MODE=true
ANTHROPIC_API_KEY=
```

`frontend/.env.example`:

```dotenv
VITE_API_URL=http://localhost:4000/api
VITE_SOCKET_URL=http://localhost:4000
```

`DATABASE_URL` remains server-only; `JWT_SECRET` signs human sessions; `CLIENT_URL`
sets CORS/socket origin; `MOCK_AGENT_MODE=true` makes an empty Claude key valid.
Never expose secrets in `VITE_*` values.

## 20. Vibe Coding Workflow

1. Inventory current routes/dependencies/fake data/screenshots; tag baseline.
2. Prompt the coding agent for exactly one subsystem and allowed files; prohibit
   unrelated UI/layout/style refactors.
3. Review `git diff`, run typecheck/tests/build, and feed exact errors back for the
   smallest repair.
4. Commit passing slices (`backend: interceptor`) rather than one huge AI change.
5. Keep `docs/STATUS.md` with done/next/blockers. Revert bad committed generations;
   restore uncommitted experiments instead of compounding unclear changes.

**Exact prompt sequence:**

1. Audit repository, report only—no edits.
2. Create backend skeleton/health/env/Helmet/CORS/error handling only.
3. Add `pg` SQL migration/seed for this plan—no ORM/frontend edits.
4. Implement tested agent CRUD/status/permission API.
5. Implement tested agent authentication and fail-closed `authorizeToolCall`.
6. Implement pure deterministic risk configuration/calculator/tests.
7. Implement ordered policy evaluator/tests.
8. Implement approval state machine and reauthorization.
9. Implement allowlisted mock registry/private executor.
10. Implement audit helper/list/filter/export/action trace.
11. Add Socket.IO persisted-event adapter.
12. Replace exactly one frontend page's fake data at a time using Query.
13. Wire three Demo buttons to the real seeded pipeline.
14. Add unit/integration tests without changing behavior.
15. Add deployment scripts/readme/env guidance only.

## 21. Git/GitHub Workflow

For one builder, prefer one `feature/hackathon-mvp` branch and tiny commits; it is
faster than multiple long-lived branches. For two teammates, use
`feature/backend-security` and `feature/frontend-integration`, with only the latter
editing established UI pages. Commit/push each passing slice and tag checkpoints,
e.g. `checkpoint-core-pipeline`. Use `git revert <sha>` for bad accepted AI commits
and `git restore` for experiments. Keep `main` demonstrable.

## 22. 24-Hour Schedule

| Time | Build and expected output | Do not work on / skip if late |
|---|---|---|
| 0–1h | Audit prototype, baseline tag, status board, provision Supabase. | UI redesign. |
| 1–3h | Backend/env/health/migration/seeds. | Socket.IO/Claude. |
| 3–6h | Registry, identity/status/permission, interceptor; prove with Postman. | Dashboard integration. |
| 6–8h | Risk/policy engines with tests. | Fancy policy editor. |
| 8–10h | Actions/approvals/executor/audit; three API paths pass. | Real integrations. |
| 10–12h | Connect Agents, Live Actions, Approvals, Audit REST pages. | UI refactors. |
| 12–14h | Overview/Risk/Decision Graph adapters + Demo endpoint. | Graph animation. |
| 14–16h | Socket.IO key events; REST fallback. | Redis/queues. |
| 16–18h | Recovery and test matrix. | Skip recovery first. |
| 18–20h | Production builds/deploy/CORS verification. | Claude integration. |
| 20–22h | Polish error/loading/demo/recording backup. | New features. |
| 22–24h | Rehearse, fix blockers, push/tag final/Postman. | Architecture changes. |

**MUST HAVE:** interceptor, status/permission checks, risk/policy, all three
decisions, approvals, mock execution, audit, three demos, REST-connected key UI.
**SHOULD HAVE:** Socket.IO, details/history, simulator, CSV, recovery, login.
**NICE TO HAVE:** Claude, editable policy forms, graph animation, rich charts.

## 23. Testing

| Test | Expected result |
|---|---|
| Active ResearchBot + `KnowledgeBase.read` | ALLOW, executes, complete audit sequence. |
| Active SalesBot + external `Email.sendBatch(38)` | REQUIRE_APPROVAL, pending, no execution yet. |
| SalesBot + `CRM.deleteCustomers(147)` | Risk 100/BLOCK; executor never called. |
| Unknown agent, paused agent, disabled agent | 401/403 blocked; no execution. |
| No permission | BLOCK before execution. |
| Unknown tool | BLOCK fail-closed and audit evidence. |
| Denied approval | Terminal denial; executor never invoked. |
| Approved approval | Reauthorize, execute, audit. |
| Mock tool exception | `FAILED` plus `ACTION_FAILED`. |
| Reversible update | Snapshot first; restore changes mock state and audits. |
| Every proposal | proposed/intercepted/risk/policy/terminal audit trail. |
| Socket client | Persisted action event updates/invalidate matching query. |

Run unit tests for risk/policy/authorization, integration tests for action/approval
transitions, `typecheck`, and production builds. Postman is the frontend-independent
end-to-end verification.

## 24. Demo Scenarios

1. **ALLOW:** ResearchBot → `KnowledgeBase.read` → risk 10–15 → allow → safe mock
   result → audit. Show active identity, permission, factors, result, audit.
2. **APPROVAL:** SalesBot → `Email.sendBatch(38)` → risk about 70 → external-email
   policy → pending approval → human approves → revalidation/execution → audit.
3. **BLOCK:** SalesBot → `CRM.deleteCustomers(147)` → risk 100 → critical/bulk-delete
   policies → block → no tool invocation → audit proof.

The judge should see the same live action across Live Actions, Decision Graph,
Approvals where applicable, and Audit Log—not a scripted frontend animation.

## 25. Recovery

Only `CRM.updateBatch` is reversible. Before its mock mutation, store a
`RecoverySnapshot` of affected records and audit `RECOVERY_CREATED`. An
authenticated `POST /api/recovery/:id/restore` restores it once, marks it restored,
and audits `RECOVERY_RESTORED`. State explicitly that this is a controlled demo
rollback, not a claim of universal external rollback.

## 26. Deployment

1. Create Supabase, run migration/seeds, add its pooled `DATABASE_URL` to backend.
2. Deploy Express to Railway; set secrets, `CLIENT_URL`, `MOCK_AGENT_MODE=true`, and
   verify `/api/health`.
3. Deploy Vite UI to Vercel; set `VITE_API_URL=https://backend/api` and
   `VITE_SOCKET_URL=https://backend` at build time, then redeploy.
4. Configure Express and Socket.IO CORS for exactly the Vercel origin; test HTTPS
   REST/socket connection.

Common failures: stale Vite build variables, failing to listen on `process.env.PORT`,
Postgres SSL/pool URL mismatch, missing protocol in CORS origin, and backend cold
starts. If cloud fails, demo locally with Postman/recorded fallback.

## 27. Postman

Create variables: `baseUrl`, `humanToken`, `salesAgentKey`, `researchAgentKey`. Run:
Health → Login (save token) → Get/Create Agents → Pause/Resume → Get Tools → Create
Action with `X-Agent-Key` → Get/Approve/Deny Approval → Get Policies → Simulate Risk
→ Get Audit → Run Demo. Add assertions for decision/status and no execution for
blocked/denied calls. This tests the full backend without the UI.

## 28. Failure / Backup Plan

| Failure | Fast fallback |
|---|---|
| Prisma breaks | Avoid it: `pg` plus SQL migrations. |
| Supabase/migration fails | Use local SQLite/in-memory seeded repository, clearly labelled demo mode. |
| Socket.IO breaks | REST refetch/poll; decisions never depend on realtime. |
| Claude fails | `MOCK_AGENT_MODE=true`; all demos still work. |
| Frontend integration fails | Show Postman + connect Demo/Approvals/Live Actions only. |
| Deployment fails | Run locally and prepare recording. |
| AI coding breaks code | Restore/revert last small commit; resume from checkpoint. |

## 29. What Not To Build

Do not build real deletion/payment/email, OAuth integrations, multi-agent
orchestration, arbitrary policy scripting, background jobs, distributed event
systems, tenancy/billing, ML risk scoring, Kubernetes, or a complete MCP proxy. Do
not rewrite the existing UI.

## 30. Definition of Done

- [ ] Existing frontend, backend, database, tools, and seeded agents run.
- [ ] Agents register and pause/resume/disable; permissions enforce server-side.
- [ ] Every proposal passes interceptor, authorization, risk, policy, and audit.
- [ ] ALLOW executes; APPROVAL waits/revalidates; BLOCK/denial never execute.
- [ ] Audits persist; action/approval/agent UI updates; key pages have real paths.
- [ ] Risk simulation, policies, decision graph, recovery demo, demo page, Postman,
  tests, production builds, and deployment or a rehearsed fallback work.

## 31. Hackathon Presentation Flow

1. **0:00–0:25:** “Agents are useful, but an LLM should not authorize consequential
   work.” Show architecture and tagline.
2. **0:25–1:10:** Agents page: ResearchBot is active with limited permission. Run
   read; show identity, risk, allow, mock result, audit.
3. **1:10–2:15:** Run SalesBot external email; show permission passed but policy
   requires approval. Approve from Approvals page; show revalidation then execution.
4. **2:15–3:10:** Run delete 147; show score 100, matching guards, blocked decision,
   and executor-not-called audit evidence.
5. **3:10–3:40:** Open Audit/Decision Graph. Close: “Claude or a mock proposes;
   deterministic AgentShield decides, records, and proves the result.”

## 32. Final Architecture Summary

```text
                 optional Claude / deterministic MockAgent
                              │ { toolName, input }
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Express security boundary                                            │
│ agent authentication → ACTIVE status → tool allowlist → permission   │
│ → deterministic risk → ordered policies → ALLOW / APPROVAL / BLOCK   │
│                                      │                 │             │
│                       approved/revalidated             └─ no execute │
│                                      ▼                               │
│                         safe mock tool executor                      │
│                                      │                               │
│                       Postgres action/risk/approval/audit            │
└──────────────────────────────────────┼──────────────────────────────┘
                                       Socket.IO + REST
                                              ▼
                  preserved React dashboard (visualization/control only)
```

### First 10 actions

1. Tag/audit untouched prototype. 2. Provision Supabase/env files. 3. Scaffold health
endpoint. 4. Add SQL migration/seed. 5. Build registry/schemas. 6. Implement agent
identity/status/permission. 7. Test risk. 8. Test policies. 9. Build interceptor,
approval, execution, audit. 10. Prove three flows in Postman before wiring React.

**Final judge promise:** An untrusted agent may propose an action. AgentShield
independently verifies who it is, what it may do, how risky it is, whether policy
permits it, and whether a human must approve it—then provides realtime, auditable
proof of the outcome.
