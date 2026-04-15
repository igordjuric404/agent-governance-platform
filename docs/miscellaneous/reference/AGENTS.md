# Ophanix - Agent Governance Platform

**Ophanix** is a private enterprise product derived from the Microsoft Agent Governance Toolkit. It provides deterministic runtime governance for AI agents — every agent action is evaluated against policy *before* execution, at sub-millisecond latency.

---

## Repository Structure

```
ai-agent-security/
├── AGENTS.md                    # This file
└── agent-governance-platform/   # Main platform (forked from Microsoft AGT)
```

### agent-governance-platform/

```
agent-governance-platform/
├── README.md                    # Platform overview and quick start
├── QUICKSTART.md                # 10-minute getting started guide
├── ARCHITECTURE.md               # System design and security model
├── packages/                     # Core SDK packages (5 languages)
├── examples/                     # Framework integration examples
├── docs/                         # Documentation and tutorials
├── demos/                        # Demo applications
├── notebooks/                    # Jupyter notebooks
├── action/                      # GitHub Actions for governance
├── scripts/                     # Utility scripts
├── benchmarks/                  # Performance benchmarks
├── fuzz/                        # Fuzzing tests (7 targets)
└── pipelines/                   # CI/CD pipelines
```

---

## Core Packages

### Multi-Language SDKs

| Package | Language | Description |
|---------|----------|-------------|
| `agent-os` | Python | Policy engine, capability model, audit logging, MCP gateway |
| `agent-mesh` | Python | Zero-trust identity, trust scoring, A2A/MCP/IATP bridges |
| `agent-governance-dotnet` | .NET | Full governance stack for .NET ecosystem |
| `agent-mesh` (TypeScript) | TypeScript | `@microsoft/agentmesh-sdk` on npm |
| `agent-mesh` (Rust) | Rust | `agentmesh` crate on crates.io |
| `agent-mesh` (Go) | Go | `github.com/microsoft/agent-governance-toolkit/sdks/go` |

### Python Package Breakdown

| Package | Purpose |
|---------|---------|
| `agent-os` | Policy engine, capability model, MCP security, audit logging |
| `agent-mesh` | Zero-trust identity (Ed25519), trust scoring (0-1000), protocol bridges |
| `agent-runtime` | Execution supervisor, privilege rings, resource limits |
| `agent-hypervisor` | 4-tier privilege rings, saga orchestration, kill switch |
| `agent-sre` | SLOs, error budgets, circuit breakers, chaos engineering |
| `agent-compliance` | OWASP verification, integrity checks, `agt` CLI |
| `agent-discovery` | Shadow AI discovery, inventory, risk scoring |
| `agent-marketplace` | Plugin lifecycle management, signing, verification |
| `agent-lightning` | RL training governance, policy rewards |
| `agent-mcp-governance` | MCP protocol governance |

### Framework Integrations (`agentmesh-integrations/`)

| Integration | Description |
|------------|-------------|
| `langchain-agentmesh` | LangChain/LangGraph adapter |
| `crewai-agentmesh` | CrewAI adapter |
| `adk-agentmesh` | Google ADK adapter |
| `openai-agents-agentmesh` | OpenAI Agents SDK middleware |
| `llamaindex-agentmesh` | LlamaIndex middleware |
| `haystack-agentmesh` | Haystack pipeline integration |
| `dify` | Dify plugin integration |
| `mcp-trust-proxy` | MCP trust proxy |
| `a2a-protocol` | Agent-to-Agent protocol |
| `copilot-governance` | GitHub Copilot governance |
| `langgraph-trust` | LangGraph trust integration |
| `openai-agents-trust` | OpenAI Agents trust |
| `pydantic-ai-governance` | Pydantic AI governance |
| `mastrar-agentmesh` | MastRa integration |
| `flowise-agentmesh` | Flowise integration |
| `aps-agentmesh` | APS integration |
| `scopeblind-protect-mcp` | Scope blind MCP protection |
| `nostr-wot` | Nostr Web of Trust |
| `openshell-skill` | OpenShell skill |
| `template-agentmesh` | Template for new integrations |

---

## Architecture

### Governance Flow

```
Agent Action ──► Policy Check ──► Allow / Deny ──► Audit Log    (< 0.1 ms)
```

### Core Components

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AGENT GOVERNANCE TOOLKIT                         │
│                                                                      │
│  ┌──────────────────────────┐     ┌──────────────────────────────┐  │
│  │      AGENT OS ENGINE     │◄───►│          AGENTMESH           │  │
│  │                          │     │                              │  │
│  │  ● Policy Engine         │     │  ● Zero-Trust Identity       │  │
│  │  ● Capability Model      │     │  ● Ed25519 / SPIFFE Certs    │  │
│  │  ● Audit Logging        │     │  ● Trust Scoring (0-1000)    │  │
│  │  ● Action Interception   │     │  ● A2A + MCP Protocol Bridge │  │
│  └────────────┬─────────────┘     └───────────────┬──────────────┘  │
│               │                                   │                  │
│               ▼                                   ▼                  │
│  ┌──────────────────────────┐     ┌──────────────────────────────┐  │
│  │     AGENT RUNTIME        │     │         AGENT SRE            │  │
│  │                          │     │                              │  │
│  │  ● Execution Rings      │     │  ● SLO Engine + Error Budgets│  │
│  │  ● Resource Limits       │     │  ● Replay & Chaos Testing    │  │
│  │  ● Runtime Sandboxing    │     │  ● Progressive Delivery     │  │
│  │  ● Termination Control   │     │  ● Circuit Breakers         │  │
│  └──────────────────────────┘     └──────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────────────┐     ┌──────────────────────────────┐  │
│  │   AGENT MARKETPLACE      │     │      AGENT LIGHTNING        │  │
│  │                          │     │                              │  │
│  │  ● Plugin Discovery      │     │  ● RL Training Governance   │  │
│  │  ● Signing & Verification│    │  ● Policy Rewards          │  │
│  └──────────────────────────┘     └──────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Agent OS Core Modules (`packages/agent-os/src/agent_os/`)

| Module | Purpose |
|--------|---------|
| `policies/` | Policy evaluation engine, YAML/OPA/Cedar backends |
| `integrations/` | Framework integrations (20+ adapters) |
| `mcp_security.py` | MCP tool poisoning, typosquatting, hidden instruction detection |
| `mcp_gateway.py` | MCP protocol gateway |
| `memory_guard.py` | Episodic memory integrity |
| `prompt_injection.py` | Prompt injection detection (12-vector) |
| `sandbox.py` | Execution sandboxing |
| `circuit_breaker.py` | Circuit breaker implementation |
| `audit_logger.py` | Append-only audit logging |
| `constraint_graph.py` | Capability constraint graphs |
| `context_budget.py` | Context window budget management |
| `credential_redactor.py` | Credential redaction |
| `reversibility.py` | Action reversibility checking |
| `secure_codegen.py` | Secure code generation |
| `security_skills.py` | Security skill definitions |
| `semantic_policy.py` | Semantic policy evaluation |
| `trust_root.py` | Trust root management |

### AgentMesh Core Modules (`packages/agent-mesh/src/agentmesh/`)

| Module | Purpose |
|--------|---------|
| `identity/` | Ed25519 identity, credentials, DID, SPIFFE/SVID |
| `trust/` | Trust scoring engine (0-1000 tiers) |
| `governance/` | Governance rules and policies |
| `lifecycle/` | Agent lifecycle (8 states) |
| `events/` | Event bus and handling |
| `transport/` | Secure transport |
| `gateway/` | Protocol gateways |
| `marketplace/` | Plugin marketplace |
| `observability/` | Telemetry and tracing |
| `storage/` | Secure storage |
| `services/` | Core services |
| `reward/` | Policy rewards |

### Agent Hypervisor (`packages/agent-hypervisor/src/hypervisor/`)

| Module | Purpose |
|--------|---------|
| `rings/` | 4-tier privilege rings (Ring 0-3) |
| `saga/` | Saga orchestration, checkpointing, fan-out |
| `session/` | Session management, isolation, SSO, intent locks |
| `security/` | Kill switch, security controls |
| `verification/` | Reversibility verification |
| `audit/` | Audit trail with commitment schemes |
| `observability/` | Event bus, causal tracing, Prometheus |
| `api/` | REST API server |

### Agent SRE (`packages/agent-sre/src/agent_sre/`)

| Module | Purpose |
|--------|---------|
| `slo/` | SLO engine, error budgets |
| `incidents/` | Incident management |
| `alerts/` | Alerting system |
| `chaos/` | Chaos engineering |
| `cascade/` | Cascading failure handling |
| `replay/` | Session replay debugging |
| `anomaly/` | Anomaly detection |
| `cost/` | Cost tracking |
| `tracing/` | Distributed tracing |
| `delivery/` | Progressive delivery |
| `certification/` | Agent certification |
| `fleet/` | Fleet management |
| `k8s/` | Kubernetes integration |
| `mcp/` | MCP observability |
| `evals/` | Evaluations |

---

## Documentation Structure

```
docs/
├── ARCHITECTURE.md              # System architecture
├── OWASP-COMPLIANCE.md          # OWASP Agentic Top 10 coverage
├── THREAT_MODEL.md              # Threat model and STRIDE analysis
├── SDK-FEATURE-MATRIX.md        # Multi-language SDK comparison
├── COMPARISON.md                # vs. other governance solutions
├── LIMITATIONS.md               # Known limitations
├── GLOSSARY.md                  # Terminology
├── modern-agent-architecture-overview.md  # Architecture deep dive
├── adr/                         # Architecture decision records
├── proposals/                   # Integration proposals (25+)
├── tutorials/                   # 30 step-by-step tutorials
├── case-studies/               # Industry case studies
├── compliance/                  # EU AI Act, NIST, SOC2 mappings
├── deployment/                  # Deployment guides
├── benchmarks/                  # Performance benchmarks
├── demos/                       # Demo scripts
├── diagrams/                    # Architecture diagrams
└── security/                    # Security documentation
```

### Key Documentation Files

| File | Description |
|------|-------------|
| `OWASP-COMPLIANCE.md` | 10/10 OWASP Agentic Top 10 coverage |
| `THREAT_MODEL.md` | Trust boundaries, STRIDE analysis |
| `BENCHMARKS.md` | Performance: <0.1ms per action |
| `SECURITY.md` | Security policy and vulnerability reporting |
| `PUBLISHING.md` | Package publishing guide |
| `CHANGELOG.md` | Version history |

---

## Examples

```
examples/
├── quickstart/                  # 5-minute quickstart examples
│   ├── langchain_governed.py
│   ├── crewai_governed.py
│   ├── autogen_governed.py
│   ├── google_adk_governed.py
│   ├── openai_agents_governed.py
│   └── retro_governed.py
├── crewai-governed/             # CrewAI integration example
├── openai-agents-governed/      # OpenAI Agents SDK example
├── smolagents-governed/         # SmolAgents example
├── openshell-governed/          # OpenShell example
├── maf-integration/             # MAF integration examples
├── mcp-trust-verified-server/  # MCP trust server
├── marketplace-governance/      # Marketplace governance
└── atr-community-rules/         # ATR community rules
```

---

## OWASP Agentic Top 10 Coverage

| Risk | ID | Control |
|------|----|---------|
| Agent Goal Hijacking | ASI-01 | Policy engine blocks unauthorized goal changes |
| Excessive Capabilities | ASI-02 | Capability model enforces least-privilege |
| Identity & Privilege Abuse | ASI-03 | Zero-trust identity with Ed25519 + ML-DSA-65 |
| Uncontrolled Code Execution | ASI-04 | Execution rings + sandboxing |
| Insecure Output Handling | ASI-05 | Content policies validate all outputs |
| Memory Poisoning | ASI-06 | Episodic memory with integrity checks |
| Unsafe Inter-Agent Comms | ASI-07 | Encrypted channels + trust gates |
| Cascading Failures | ASI-08 | Circuit breakers + SLO enforcement |
| Human-Agent Trust Deficit | ASI-09 | Full audit trails + flight recorder |
| Rogue Agents | ASI-10 | Kill switch + ring isolation + anomaly detection |

---

## Trust Scoring

AgentMesh assigns trust scores on a 0–1000 scale:

| Score | Tier | Meaning |
|-------|------|---------|
| 900–1000 | Verified Partner | Cryptographically verified, long-term trusted |
| 700–899 | Trusted | Established track record, elevated privileges |
| 500–699 | Standard | Default for new agents with valid identity |
| 300–499 | Probationary | Limited privileges, under observation |
| 0–299 | Untrusted | Restricted to read-only or blocked |

---

## Execution Privilege Rings

| Ring | Level | Description |
|------|-------|-------------|
| Ring 0 | Admin | Full tool access — trusted orchestrators |
| Ring 1 | Standard | Scoped tool access — most production agents |
| Ring 2 | Restricted | Read-only + approved writes — new/untested agents |
| Ring 3 | Sandboxed | No external access — training and testing |

---

## Testing & Security

| Tool | Coverage |
|------|----------|
| CodeQL | Python + TypeScript SAST |
| Gitleaks | Secret scanning |
| ClusterFuzzLite | 7 fuzz targets |
| Dependabot | 13 ecosystems |
| OpenSSF Scorecard | Weekly security scoring |
| Pre-commit hooks | Linting, formatting |

---

## Development Workflows

### GitHub Actions

- `governance-attestation/` - Attest agent compliance
- `security-scan/` - Scan for vulnerabilities
- ESRP publishing pipeline

### Scripts

```bash
scripts/
├── check_gov.py                  # Verify governance installation
├── check_dependency_confusion.py  # Dependency confusion check
├── check_vendor_imports.py       # Vendor import checks
├── generate_sbom.py              # SBOM generation
└── security_scan.py             # Security scanning
```

---

## Supported Frameworks

| Framework | Integration Type |
|-----------|-----------------|
| Microsoft Agent Framework | Native Middleware |
| Semantic Kernel | Native (.NET + Python) |
| AutoGen | Adapter |
| LangChain / LangGraph | Adapter |
| CrewAI | Adapter |
| OpenAI Agents SDK | Middleware |
| Google ADK | Adapter |
| LlamaIndex | Middleware |
| Haystack | Pipeline |
| Dify | Plugin |
| AWS Bedrock | Adapter |
| Azure AI Foundry | Deployment Guide |
| Anthropic | Integration proposal |
| Mistral | Integration proposal |
| SmolaAgents | Example |
| MetaGPT | Integration proposal |

---

## Key Statistics

- **Policy evaluation latency:** < 0.1 ms per action
- **OWASP coverage:** 10/10 risks covered
- **Tests:** 9,500+
- **Supported languages:** Python, TypeScript, .NET, Rust, Go
- **Framework integrations:** 20+
- **Tutorials:** 30
- **Integration proposals:** 25+
