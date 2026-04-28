## Product One‑Pager: Agent Governance Toolkit (AGT)

- **What it is**: An open-source, MIT-licensed **runtime governance layer for autonomous AI agents** (Microsoft-maintained, currently **Public Preview**) that deterministically enforces policy on **agent actions** (tool calls, resource access, inter-agent messages) *before* execution.
- **What it is not**: A prompt guardrails / content moderation product; it governs what agents *do*, not what they *say*.

### Goal
Enable teams to ship autonomous agents into production with **deterministic, auditable control** over agent capabilities and behavior—without adding meaningful latency—so security/compliance requirements are met without stalling delivery.

### Definition of success
- **Deterministic enforcement**: **0.00% Safety Violation Rate** on a published 60‑prompt red-team benchmark (baseline prompt-only safety: **26.67% SVR**).
- **Negligible overhead**: Published benchmarks show **~0.011 ms** p50 for single‑rule policy eval, **~0.030 ms** p50 for 100‑rule eval, and **~0.103 ms** p50 for full kernel enforcement per action.
- **Coverage & alignment**: Demonstrated mapping to **OWASP Top 10 for Agentic Applications (2026)** with **10/10 covered**, with links to controls and components.
- **Adoptable in practice**: Teams can integrate governance into an existing agent framework quickly (minutes, not weeks) using adapters/middleware, and can verify posture via CLI (`agt verify`, `agt doctor`).

### Backstory (why this, why now)
Agent systems are moving from “chat” to **action**: writing files, calling APIs, operating on data, and coordinating across multiple agents. The dominant safety pattern—“tell the model to follow rules”—is inherently probabilistic and can be bypassed via jailbreaks, social engineering, or tool misuse.

AGT treats agent actions like syscalls: it places a **policy enforcement point** between the agent framework and the side effects. This aligns with emerging expectations in 2026 (OWASP Agentic Top 10, NIST AI RMF, upcoming regulatory scrutiny) where organizations need **audit trails, least privilege, and runtime controls**—not just best-effort prompts.

### Must-have items (v1.0 scope)
- **Action interception + deterministic policy engine** for tool calls/resource access/messages with deny/allow decisions before execution.
- **Policy-as-code** that teams can review and ship (supports YAML and common policy ecosystems like OPA/Rego and Cedar).
- **Auditability**: Structured audit logs of attempted actions and decisions; exportable to standard observability pipelines.
- **Framework-agnostic integrations** (middleware/adapters) so it works with existing stacks rather than requiring a new framework.
- **Verification UX**: A CLI that can “doctor/verify/lint” governance posture for CI/CD and stakeholder reporting.
- **Documented security boundaries** and a recommended **defense-in-depth** deployment pattern (governance + container/infra isolation).

### Out-of-scope items (explicitly not included)
- **Model safety / content moderation** (use a separate layer such as content safety filters/guardrails for outputs).
- **OS kernel or hardware isolation** (AGT is application-layer; use containers/VMs for OS-level separation).
- **Reasoning correctness**: AGT does not determine whether an allowed action is “wise,” accurate, or hallucination-free.
- **Outcome verification**: Audit logs record attempts/decisions; they do not guarantee external-world success.
- **Workflow-level intent policing (yet)**: Individually-allowed actions can still compose into harmful workflows; this is a known limitation with mitigations and roadmap work.

### Competition / alternatives
- **Prompt guardrails + “safety prompts”**: Easy to start, but probabilistic and bypassable; offers weak guarantees for action security.
- **Content moderation tools**: Useful for output filtering, but don’t stop an agent from taking unsafe actions.
- **Generic policy engines (e.g., OPA)**: Strong building block, but typically require bespoke wiring into agent frameworks and don’t provide an end-to-end agent governance stack (identity/trust/audit/adapters).
- **Sandboxing only (containers/gVisor/Kata)**: Helps with isolation, but doesn’t give fine-grained, policy-driven control over *which* actions are allowed or produce governance-grade audit semantics.

### Key timing elements
- **Immediate (pilot)**: Run a proof-of-concept by wrapping one existing agent workflow with AGT policies, measuring blocked actions + audit quality, and validating latency impact (expected to be negligible relative to LLM calls).
- **Near-term (hardening)**: Expand policies/templates for your highest-risk tools and data paths; wire audit events into your logging/SIEM and define SLOs for agent reliability.
- **Before broad rollout**: Decide on your required isolation posture (per-agent containers/VMs, network policies) and validate compliance mappings relevant to your domain.
- **Risk to plan for**: AGT is in **Public Preview** and may introduce breaking changes before GA; treat early adoption as an iterative integration rather than a one-time install.

### References (repo)
- `README.md` (overview + capabilities + install)
- `QUICKSTART.md` (10-minute integration walkthrough)
- `docs/OWASP-COMPLIANCE.md` (OWASP Agentic Top 10 mapping)
- `BENCHMARKS.md` + `packages/agent-os/modules/control-plane/benchmark/README.md` (latency/throughput and safety violation benchmark methodology)
- `docs/LIMITATIONS.md` (design boundaries and mitigations)
