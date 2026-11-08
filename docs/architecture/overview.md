# Architecture and MVP scope

Status: implementation baseline; no application or containment controls exist yet.

## Scope

The first usable release runs one versioned Juice Shop scenario with one Red
agent and one Blue agent. A project member starts a run, reviews evidence,
approves an in-scope remediation, and sees a health-checked retest result.
Multi-agent teams, Kubernetes, Terraform, Ansible, and Grafana are deferred.

## Components and ownership

- React and TypeScript provide the run dashboard and approval interface.
- Django and Django REST Framework own authentication, project authorization,
  scenario versions, run state, policies, approvals, findings, and audit events.
- PostgreSQL stores durable business state. Large evidence artifacts use a
  storage interface with project-scoped access and retention controls.
- A separate Python worker coordinates agent workflows outside HTTP requests.
  LangGraph is a candidate adapter, not a dependency of domain models.
- A narrow sandbox runner provisions and destroys arenas and executes validated
  tool requests. Only this service holds arena infrastructure credentials.
- Model-provider access stays in the trusted control plane. Arena processes
  receive neither provider keys nor database credentials.

Start with a modular Django application and separate worker and runner processes.
The worker calls domain services rather than writing around authorization or
state-transition rules. Agents never write directly to PostgreSQL.

## Proposed repository layout

```text
frontend/src/{app,features,components,api}/
backend/config/settings/
backend/apps/{accounts,projects,scenarios,runs,findings,policies,audit}/
backend/orchestration/{agents,workflows,tools,providers}/
backend/tests/
sandbox/{runner,policies,tests}/
scenarios/juice-shop/
contracts/openapi/
infra/docker/
docs/{architecture,decisions,security,development}/
.github/workflows/
```

Keep migrations with their Django apps. Keep scenario manifests and pinned
target image references separate from trusted platform deployment definitions.
Publish an OpenAPI contract for the frontend. Add dependency manifests and lock
files when scaffolding, with environment examples containing placeholders only.

## Domain records

Every resource belongs to a project; every API lookup and artifact download must
enforce membership and the required role. Separate viewer, operator, and approver
permissions. Decide whether operators may approve their own actions before MVP.

- ScenarioVersion: immutable target image digest, tool allowlist, network scope,
  resource limits, health checks, and rules of engagement.
- Run: scenario version, policy version, initiator, lifecycle state, deadline,
  cancellation flag, budget, and cleanup state.
- ToolExecution: run, agent identity, validated arguments, action digest,
  execution status, idempotency key, deadline, and evidence references.
- Approval: exact action digest, approver, decision, expiry, and consumption time.
- Finding and Retest: evidence, affected target, remediation, test definition,
  target health, and outcome. Tool failure produces an inconclusive retest.
- AuditEvent: project, run, actor, ordered sequence, timestamp, event type,
  correlation ID, and redacted payload.

## Run lifecycle and reliability

Normal progression is queued -> provisioning -> running -> completed. A running
run may enter awaiting_approval and resume after an approval. Active states may
transition to cancelling -> cancelled, failed, or timed_out. Completion means
the workflow finished, not necessarily that all findings were fixed.

Cleanup has its own pending -> running -> succeeded/failed status for every
terminal run. Do not report an arena as removed before verifying removal. Alert
on failed cleanup and retry it through an independent reconciliation process.

Use transactional state transitions, worker leases, heartbeats, and a durable job
mechanism. Redelivery must not repeat side effects: reconcile uncertain tool
outcomes before retrying. Persist cancellation before terminating processes;
recheck cancellation and policy at dispatch. Approval waits retain deadlines.

## Action contract

An action request includes project/run IDs, agent identity, registered tool name,
schema-validated arguments, target identity, policy version, deadline, resource
budget, idempotency key, and an approval reference when required.

The policy gate denies unknown tools, arbitrary host commands, expired requests,
out-of-scope targets, and exhausted budgets. Out-of-scope actions cannot be
approved; changing scope requires a new reviewed scenario version and run.
Approvals bind to immutable action digests and expire. The runner independently
checks a short-lived authorization bound to that digest and run, and refuses
replays. Resolve targets against the arena inventory at execution time.

Tool results contain execution status, bounded stdout/stderr, timestamps,
artifact references, and structured errors. Sanitize artifacts for UI rendering.
Treat all tool output as untrusted data, never as policy instructions.

## Delivery order

1. Architecture, threat model, and documentation alignment (this baseline).
2. Frontend/backend scaffolding, PostgreSQL, development Compose, and CI.
3. Authenticated project-scoped APIs, run state, approvals, and audit persistence.
4. Sandbox runner and isolation acceptance tests.
5. One bounded Red/Blue workflow with remediation approval and valid retesting.

Before scaffolding, record dependency versions, authentication approach, job
backend, and worker deployment model in decision records. Before autonomous
execution, satisfy the [security acceptance gates](../security/threat-model.md).
