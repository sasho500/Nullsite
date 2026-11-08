# Threat model and sandbox acceptance gates

Status: required design controls; none are implemented or verified yet.

## Assets and trust boundaries

Protect the host, control-plane credentials, model-provider keys, project data,
approval integrity, audit evidence, and other runs. Assume targets are vulnerable
and that agent-selected code, target pages, logs, and tool output are hostile.
An authenticated project user may also attempt unauthorized cross-project access.

The boundaries are browser to API; API to worker; worker to policy gate and
runner; runner to arena; arena to host and other networks; and control plane to
external model providers. Each boundary needs authenticated, authorized, bounded
requests. A model response never grants permission.

## Required controls

| Threat | Required mitigation |
|---|---|
| Host takeover or container escape | Disposable dedicated VM or worker host for hostile execution; patched runtime; no privileged containers, host namespaces, host bind mounts, or arena Docker socket access |
| Network escape, SSRF, lateral movement | Per-run networks; deny external egress and access to host, metadata endpoints, database, control plane, and other runs; cover IPv4, IPv6, DNS, and multi-network bypasses |
| Excess privileges | Non-root execution, dropped capabilities, no-new-privileges, syscall filtering, read-only filesystems where feasible, and explicitly bounded writable storage |
| Prompt injection or approval bypass | Typed tool allowlist, runner-side authorization, immutable approval digest, expiry, single consumption, execution-time scope checks, and default denial |
| Resource exhaustion | CPU, memory, process, disk, output, wall-clock, action-count, request-rate, and model-cost budgets |
| Secret exfiltration | No platform/provider secrets in arenas; trusted model gateway with restricted destinations; redact evidence before external transmission |
| Cross-project access | Project-scoped queries and object authorization on API, jobs, approvals, events, and artifacts |
| Audit tampering or log injection | Restricted append-only audit writer, separate mutation privileges, structured escaped events, redaction, retention, and protected export; database storage alone is insufficient |
| Unsafe remediation | Human approval for disruptive in-scope changes, disposable targets, preserved patch evidence, rollback information, and health-checked retests |
| Supply-chain compromise | Reviewed scenario manifests, target image digest pinning, dependency locks, scanning, and separation of provisioning from offline run execution |

Compose internal networks are one containment layer. Do not attach arenas to
an egress-capable second network or publish target ports by default. Any required
control channel must expose a narrow authenticated protocol without general
forwarding. Collect telemetry through bounded channels; arena-originated data
must not imply control-plane network access.

The runner is a privileged trusted component: restrict its infrastructure API
access to dedicated arena resources and prevent users or agents from submitting
arbitrary Compose files, image builds, mounts, or daemon operations.

An emergency stop must terminate execution and revoke run permissions without
cooperation from the model. Cleanup must remove containers, networks, volumes,
temporary credentials, and artifacts according to retention policy.

## Acceptance gates before autonomous execution

Run these tests later on a disposable test host; this document does not claim
they have been performed.

1. Verify permitted target traffic works and prohibited internet, DNS, host,
   metadata, control-plane, database, and cross-run traffic fails, for all enabled
   IP families. Repeat under the actual deployment configuration.
2. Verify arena processes cannot access the runtime socket, host files, provider
   keys, or infrastructure credentials, and cannot elevate privileges.
3. Reject unknown tools, changed approval arguments, expired/replayed approvals,
   cross-project requests, stale scope, and requests after cancellation.
4. Exercise process/output floods, timeouts, worker crashes, and uncertain tool
   completion. Verify bounded resources and no duplicate side effects.
5. Exercise cancellation and emergency stop during provisioning, execution, and
   approval waits. Verify cleanup or a visible failed-cleanup alert and retry.
6. Confirm persisted audit events, secret redaction, safe evidence rendering, and
   denial of audit mutation by ordinary application roles.
7. Confirm unhealthy targets and failed tools yield inconclusive retests rather
   than resolved findings.

## Reference material

- [Docker Engine security](https://docs.docker.com/engine/security/)
- [Compose network isolation](https://docs.docker.com/reference/compose-file/networks/)
- [Compose service security settings](https://docs.docker.com/reference/compose-file/services/)

These controls reduce risk; container configuration alone does not establish a
safe boundary for hostile code. Revisit this model when adding tools, targets,
multi-agent execution, or deployment environments.
