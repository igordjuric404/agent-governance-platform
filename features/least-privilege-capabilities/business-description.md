# Least-Privilege Capabilities — Business-Oriented Description

## Executive Summary

**Least-Privilege Capabilities** is Ophanix's implementation of the security principle that every AI agent should operate with **exactly the permissions it needs — nothing more**. This isn't just a best practice; it's a fundamental security control that limits both the blast radius of incidents and the potential for misuse.

## The Problem We're Solving

### The Overprivileged Agent Problem

Most AI agent deployments start with a simple question: "What should this agent be able to do?" The typical answer? Grant broad permissions "to be safe" or "to not get in the way."

This creates dangerous overprivileged agents:

| Agent Type | Common Overprivilege | Risk |
|------------|---------------------|------|
| Web search agent | Full file system access | Could read/delete any file |
| Data analyzer | Production database write | Could corrupt data |
| Customer service bot | Financial system access | Could make unauthorized changes |
| Code assistant | Production deployment | Could push broken code |

### The Cost of Overprivilege

- **Data breaches** — Overprivileged agents with access to sensitive data
- **Ransomware** — Malware spreading through agent permissions
- **Compliance violations** — Accessing data beyond role requirements
- **Accidental damage** — Simple errors becoming catastrophes

## Value Proposition

### What Least-Privilege Capabilities Delivers

| Capability | Without Least-Privilege | With Least-Privilege |
|------------|----------------------|---------------------|
| Permission scope | Everything | Exactly what's needed |
| Incident blast radius | Entire system | Agent's limited scope |
| Compliance evidence | Hard to prove | Clear capability boundaries |
| Agent isolation | Weak | Strong containment |
| Audit clarity | Ambiguous | Precise capability mapping |

### Risk Reduction

**Without least-privilege:**
- One compromised agent = potential full system access
- Average breach cost: $4.45M

**With least-privilege:**
- One compromised agent = limited to defined capabilities
- Typical incident scope: Minimal, contained

## Use Cases

### 1. Financial Services — Trading Agents

**Challenge:** Trading algorithms need market data access but shouldn't touch customer accounts.

**Solution:**
```yaml
trading-agent:
  capabilities:
    - market_data.read
    - order.submit:own_account
    - risk_model.read
  denied:
    - customer_data.read
    - account.modify
    - transfer.execute
```

**Result:** Agent can trade within its mandate, cannot access customer data.

### 2. Healthcare — Patient Data Agents

**Challenge:** AI assistants handling patient inquiries shouldn't see full medical records.

**Solution:**
```yaml
patient-inquiry-agent:
  capabilities:
    - patient.search:by_name
    - appointment.read
    - general_faq.read
  denied:
    - medical_records.read
    - lab_results.access
    - prescription.modify
```

**Result:** HIPAA compliance through technical enforcement, not policy.

### 3. Software Development — Code Generation

**Challenge:** AI coding assistants should improve productivity without risking production.

**Solution:**
```yaml
code-gen-agent:
  capabilities:
    - code.read:repository
    - code.write:feature_branch
    - test.execute
    - pr.create
  denied:
    - deploy.execute
    - production.access
    - secrets.read
    - main_branch.write
```

**Result:** Developer productivity with zero production risk.

## ROI Analysis

### Cost of Not Using Least-Privilege

| Incident Type | Average Cost |
|---------------|--------------|
| Data breach from overprivileged agent | $4.45M |
| Compliance violation fine | $1.5M average |
| System corruption from bad deploy | $500K+ |
| Incident response (overprivileged) | 3x normal |

### Least-Privilege Investment

| Component | Cost |
|-----------|------|
| Initial capability definition | 1-2 days per agent type |
| Ongoing maintenance | Minimal |
| Technology | Part of Ophanix platform |

**ROI:** One prevented incident exceeds years of implementation cost.

## Compliance Alignment

Least-Privilege Capabilities directly supports:

| Regulation | Requirement | Coverage |
|------------|-------------|----------|
| EU AI Act Art. 14 | Human oversight | Conditional capabilities require approval |
| HIPAA | Minimum necessary | Explicit capability scoping |
| SOC 2 | Access control | Capability-based access decisions |
| PCI-DSS | Limited access | Denied capabilities prevent card data |
| ISO 27001 | Need-to-know | Explicit capability grants |

## Competitive Differentiation

| Feature | Traditional IAM | Ophanix Least-Privilege |
|---------|-----------------|------------------------|
| Scope | Role-based | Agent-specific |
| Granularity | Coarse | Fine-grained capability |
| Enforcement | Policy suggestion | Technical enforcement |
| Default state | Allow (with exceptions) | Deny (with grants) |
| Resource constraints | Limited | Full support |

## Implementation Approach

### Phase 1: Discovery (1 week)
- Identify all agent types
- Document current permissions
- Map to minimum required capabilities

### Phase 2: Definition (1-2 weeks)
- Define capabilities per agent type
- Establish resource constraints
- Set temporal conditions where needed

### Phase 3: Enforcement (2 weeks)
- Deploy capability registry
- Integrate with policy engine
- Begin enforcement

### Phase 4: Optimization (ongoing)
- Monitor denials, adjust as needed
- Add capabilities for legitimate needs
- Refine based on operational data

## Customer Success Metrics

Organizations implementing Least-Privilege Capabilities typically achieve:

- **95%+** of agents with strictly scoped capabilities
- **70%** reduction in incident blast radius
- **100%** capability coverage for compliance
- **Near-zero** privilege escalation incidents

## Key Pitch Points

1. **"Contain the blast radius"** — Even if an agent is compromised, damage is limited to its defined scope

2. **"Compliance made technical"** — Demonstrate HIPAA/SOC2 compliance through actual enforcement, not policies

3. **"Zero-trust agent access"** — Every capability explicitly granted, everything else denied by default

4. **"Developer velocity with safety"** — Give developers powerful AI tools without production risks

---

**Ready to enforce the principle of least privilege across your AI agent fleet?** Let's discuss your requirements.
