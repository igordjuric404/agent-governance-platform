# Privilege Rings — Non-Technical Overview

## What Are Privilege Rings?

If sandboxing is like putting an agent in a fenced yard, **Privilege Rings** are like having different levels of security clearance — from "public access" to "top secret."

Inspired by CPU hardware privilege levels (Ring 0-3 in Intel/AMD processors), Ophanix implements **4-tier privilege rings** that define exactly what an agent can do based on its trust level and role.

## The Four Rings

### Ring 0 — Administrator
**Full access to everything.** Only the most trusted orchestrator agents operate here. Think of it like the CEO who has keys to everything.

### Ring 1 — Standard User
**Normal operational access.** Most production agents work here with scoped but broad permissions. Like a department manager with access to their department's resources.

### Ring 2 — Restricted User
**Limited access with controls.** New or untested agents operate here. Like a new employee on probation — they can do their job but can't access sensitive systems.

### Ring 3 — Sandboxed
**Minimal access, maximum safety.** Training environments, testing, or truly untrusted code. Like a visitor badge that only allows access to the lobby.

## Why Four Rings?

One ring would be too simple — you'd have to choose between "everything" and "nothing." Three rings aren't enough to express the nuances of enterprise AI governance.

Four rings give you the granularity to:

- Give trusted orchestrators full power
- Limit production agents appropriately
- Closely monitor new agents
- Isolate completely untrusted code

## Real-World Analogy

Think of privilege rings like a **hospital**:

- **Ring 3 (Sandboxed):** Patients and visitors — can only access public areas
- **Ring 2 (Restricted):** New staff on orientation — supervised, limited access
- **Ring 1 (Standard):** Regular staff — can do their jobs but don't have keys to everywhere
- **Ring 0 (Admin):** Chief surgeon — has access to all areas for emergencies

## Key Benefits

1. **Granular Access Control** — Four distinct levels instead of binary allow/deny
2. **Risk Stratification** — Higher-risk operations require higher rings
3. **Progressive Trust** — Agents can "level up" as they prove themselves
4. **Automatic Enforcement** — Ring boundaries enforced by the system, not policies

## The Bottom Line

Privilege Rings provide a **structured hierarchy of trust** for AI agents. Just as military clearances and corporate access levels ensure people only access what they need, Privilege Rings ensure AI agents operate at appropriate trust levels — with clear paths to elevation when earned.
