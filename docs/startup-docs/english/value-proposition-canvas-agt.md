## Value Proposition Canvas — Agent Governance Toolkit (AGT)

![Value Proposition Canvas — AGT](./value-proposition-canvas-agt.svg)

If your editor/preview won’t render the SVG, open `value-proposition-canvas-agt.html` in a browser (it inlines the same diagram).

### Scope, customer segment, and assumptions (stated up front)

- **Customer segment (primary)**: **Enterprise platform + security engineering teams** that are responsible for shipping **agentic applications** (agents that can call tools, access data, and take real-world actions) into production.
- **Key stakeholders inside this “customer”**: platform engineering (integration + latency), security engineering (least privilege + threat control), compliance/risk (evidence + mappings), SRE/ops (reliability + incident containment).
- **Assumptions (because we only have repo/docs + competitor notes, not customer interviews)**:
  - The buyer already runs agents in a framework (e.g., LangChain, OpenAI Agents SDK, AutoGen, CrewAI) and needs **runtime action governance**, not just content filtering.
  - The buyer cares about **auditability and deterministic guarantees** (OWASP-style controls, policy-as-code), and is willing to deploy defense-in-depth (e.g., per-agent containers/VMs) where required.
  - The buyer may still need a **separate content-safety/DLP layer** and/or **continuous red-teaming/evals** (AGT is explicit that it is not primarily a prompt/output moderation product).

### Evidence anchors from the available data (why this isn’t generic)

- **Deterministic enforcement benchmark**: **0.00% Safety Violation Rate** vs **26.67%** for a prompt-only baseline on a published 60‑prompt red-team dataset.
- **Governance overhead benchmarks (p50)**: ~**0.011 ms** (single-rule policy eval), ~**0.030 ms** (100-rule eval), ~**0.103 ms** (full kernel path).
- **Scale benchmark**: ~**47K ops/sec** at **1,000 concurrent agents** (policy path sustained).
- **Coverage claim in docs**: **OWASP Agentic Top 10 (2026) — 10/10 covered** via mapped controls.

---

### 1) Customer Profile

#### Customer Jobs (center)

- **J1 — Ship agentic apps with controlled side effects**: allow autonomy where safe, block/contain where not.
- **J2 — Enforce least-privilege for tools and data**: define “what agents can do” as policy, not prompts.
- **J3 — Produce audit evidence stakeholders accept**: security reviews, compliance attestations, incident forensics.
- **J4 — Operate and recover safely**: stop runaway agents, prevent cascades, roll back multi-step workflows.
- **J5 — Integrate governance into existing stacks**: minimal rework across languages/frameworks/clouds.
- **J6 — Reduce supply-chain risk from tools/MCP**: avoid tool poisoning, typosquatting, hidden instructions, rogue agents.

#### Pains

- **P1 — Prompt-only safety is bypassable**: jailbreaks/social engineering lead to unsafe tool calls and policy violations.
- **P2 — Overprivileged tools/capabilities**: agents accidentally (or maliciously) delete/alter data, exfiltrate secrets, or escalate permissions.
- **P3 — Audit/compliance friction**: hard to prove “what happened” and “what is allowed” for agents; approvals stall launches.
- **P4 — Multi-agent identity & privilege abuse**: spoofed agents, unclear trust, unsafe inter-agent comms.
- **P5 — MCP/tool supply-chain risk**: poisoned tool definitions, hidden instructions, or malicious MCP servers.
- **P6 — Cascading failures & runaway behavior**: one failure propagates across tools/agents; hard to contain and debug.
- **P7 — Governance perceived as expensive**: fear of latency overhead, fragility, or massive integration effort.
- **P8 — “One-box” buyer expectations**: orgs expect TRiSM breadth (DLP/PII, content safety, red teaming, GRC packaging) and may see action-governance-only as incomplete unless paired.

#### Gains

- **G1 — Deterministic action control**: clear allow/deny enforcement for every agent action (not “best effort”).
- **G2 — Low overhead at scale**: governance is sub-ms and high-throughput so it’s operationally invisible.
- **G3 — Reduced blast radius**: least privilege + approvals + containment controls reduce worst-case impact.
- **G4 — Faster approvals**: strong audit trails + recognized mappings (OWASP/NIST/EU AI Act-style) shorten review cycles.
- **G5 — Safe multi-agent collaboration**: identity + trust gating + secure communications enable delegation without chaos.
- **G6 — Fit to existing ecosystems**: works across frameworks, languages, and clouds without lock-in.
- **G7 — Reliability & debugability**: SLOs, replay, circuit breakers, and runbooks for agent fleets.
- **G8 — Vendor independence**: open-source + standard policy formats reduce long-term lock-in risk.

---

### 2) Value Map

#### Products & Services (center)

- **PS1 — Deterministic policy engine (Agent OS)**: evaluates every agent action before execution; policy-as-code (YAML + common policy ecosystems).
- **PS2 — Capability model / least-privilege enforcement**: explicit allowlists/denylists and scoped grants.
- **PS3 — Zero-trust agent identity (AgentMesh)**: cryptographic identities, trust scoring, delegation constraints, secure inter-agent comms.
- **PS4 — Execution supervision (Runtime/Hypervisor)**: privilege rings, resource limits, sagas/compensation, kill switch.
- **PS5 — Agent SRE toolkit**: SLOs, error budgets, circuit breakers, replay debugging, chaos testing primitives.
- **PS6 — MCP security scanner/gateway**: detects tool poisoning/typosquatting/hidden instructions in MCP definitions.
- **PS7 — Discovery + governance visibility**: inventory/“shadow AI” discovery, lifecycle hooks, governance dashboards (where deployed).
- **PS8 — Unified CLI & verification**: posture checks, OWASP coverage verification, policy linting, integrity verification.

#### Pain Relievers

- **PR1 — Deterministic pre-execution enforcement** blocks disallowed tool calls (addresses “prompt bypass → action” failures).
- **PR2 — Deny-by-default + scoped capabilities** reduces overprivilege and accidental damage.
- **PR3 — Approval workflows for high-risk actions** adds explicit human oversight where required.
- **PR4 — Cryptographic identity + trust scoring** reduces spoofing/privilege abuse and unsafe delegation.
- **PR5 — Secure inter-agent comms + trust gates** reduce cross-agent compromise propagation.
- **PR6 — Containment controls** (rings, resource limits, kill switch, saga rollback) cap blast radius.
- **PR7 — Agent SRE circuit breakers & SLO enforcement** prevent cascading failures and automate “stop the bleeding”.
- **PR8 — MCP scanner/gateway** reduces tool/MCP supply-chain attack surface.
- **PR9 — Performance transparency (benchmarks)** reduces the “governance will slow us down” objection.

#### Gain Creators

- **GC1 — Published benchmark outcomes** (e.g., 0% safety violations on a benchmark where prompt-only baselines violate policy materially) increase executive confidence.
- **GC2 — Sub-ms latency + high throughput** makes governance practical for real-time tool use.
- **GC3 — OWASP Agentic Top 10 mapping (10/10) + verification tooling** turns “security posture” into an artifact.
- **GC4 — Broad framework + language integrations** lowers adoption friction (teams keep their stack).
- **GC5 — Vendor independence (MIT, deploy-anywhere)** supports long-term platform bets and avoids lock-in.
- **GC6 — Fleet visibility + discovery** supports governance at scale beyond a single agent instance.
- **GC7 — Defense-in-depth guidance + known limitations transparency** helps teams design correct layered architectures.
- **GC8 — Extensibility**: policy-as-code and open source allow tailoring to domain-specific controls.

---

### 3) Explicit Fit (mapped precisely)

#### Pains → Pain Relievers (why the value map matches the customer profile)

| Customer Pain | Primary Pain Relievers (and why) |
|---|---|
| **P1 Prompt-only safety is bypassable** | **PR1** (deterministic pre-execution checks stop unsafe actions even if the model is “tricked”); **PR2** (deny-by-default narrows what “bypass” can do); **PR6** (containment if something slips). |
| **P2 Overprivileged tools/capabilities** | **PR2** (scoped capabilities), **PR3** (approvals for sensitive actions), **PR6** (rings/resource limits/kill switch to cap damage). |
| **P3 Audit/compliance friction** | **PR1** + **PR4** (every action is evaluated with an attributable identity), plus **PS8** (verification outputs) to produce reviewable artifacts. |
| **P4 Identity & privilege abuse (multi-agent)** | **PR4** (cryptographic identity + trust scoring) and **PR5** (secure inter-agent comms + trust gates). |
| **P5 MCP/tool supply-chain risk** | **PR8** (MCP scanner/gateway) plus **PR2** (capability scoping limits what a poisoned tool can do). |
| **P6 Cascading failures & runaway behavior** | **PR7** (circuit breakers/SLOs stop cascades), **PR6** (kill switch + sagas/rollback), **PR1** (policy blocks prevent unsafe expansions). |
| **P7 Governance perceived as expensive** | **PR9** (benchmarks + transparency reduce uncertainty), **GC2** (sub‑ms overhead makes it operationally feasible), **GC4** (adapters/SDKs reduce integration cost). |
| **P8 “One-box” expectations (content safety, DLP, red teaming, GRC packaging)** | **Partial fit by design**: AGT covers action-governance strongly (**PR1/PR2/PR4/PR6/PR7/PR8**), but **does not replace** content moderation/red-teaming suites. The “fit” here is via **GC7** (explicit layered architecture guidance) and integration-ready posture—pair with content safety/evals when required. |

#### Gains → Gain Creators

| Customer Gain | Gain Creators that deliver it (and why) |
|---|---|
| **G1 Deterministic action control** | **GC1** (demonstrated outcomes), anchored by **PS1/PR1** (deterministic enforcement). |
| **G2 Low overhead at scale** | **GC2** (published latency/throughput) + architecture built for pre-execution checks. |
| **G3 Reduced blast radius** | **PR2/PR3/PR6** (least privilege + approvals + containment) reinforced by **PS4**. |
| **G4 Faster approvals** | **GC3** (OWASP mapping + verification artifacts) + auditable decision logs and identity context. |
| **G5 Safe multi-agent collaboration** | **PS3** → **PR4/PR5** (identity, trust, secure A2A). |
| **G6 Fit to existing ecosystems** | **GC4** (framework + language integrations) reduces migration effort. |
| **G7 Reliability & debugability** | **PS5/PR7** (SLOs, circuit breakers, replay/chaos) support ops workflows. |
| **G8 Vendor independence** | **GC5** (MIT + deploy-anywhere) + policy-as-code portability. |

#### Jobs → Products & Services (what the customer “hires” AGT to do)

| Customer Job | Products & Services that enable it |
|---|---|
| **J1 Ship controlled side effects** | **PS1** (action interception + policy) + **PS4** (execution supervision) |
| **J2 Enforce least privilege** | **PS2** (capabilities) + **PS1** (policy-as-code) |
| **J3 Produce audit evidence** | **PS1** (policy decisions) + **PS8** (verification + integrity tooling) |
| **J4 Operate & recover safely** | **PS5** (SRE) + **PS4** (kill switch/sagas/rings) |
| **J5 Integrate into existing stacks** | **PS8** (CLI workflows) + adapters/SDKs across major frameworks/languages |
| **J6 Reduce tool/MCP supply-chain risk** | **PS6** (MCP scanner/gateway) + **PS7** (discovery) |

---

### 4) Fit boundaries (important so the canvas stays honest)

- **Content safety / DLP / output moderation** is not AGT’s core scope; if the customer’s *primary* job is “prevent PII leakage in LLM outputs,” the fit requires pairing AGT with a content-safety layer.
- **Workflow-level malicious sequences** can exist even when each individual action is allowed; customers must design policies and guardrails with this limitation in mind (AGT documents this boundary and recommended mitigations).
- **OS-level isolation** requires containers/VMs; AGT is an application-layer enforcement point (best used as part of defense-in-depth).

