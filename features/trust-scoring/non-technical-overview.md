# Trust Scoring — Non-Technical Overview

## What Is Trust Scoring?

Imagine a credit score for AI agents. Just as credit scores help banks decide whether to trust you with a loan, **Trust Scoring** helps organizations decide how much to trust their AI agents — and what permissions to give them.

Ophanix assigns every agent a **Trust Score from 0 to 1000**. This score isn't arbitrary — it's calculated based on real behavior: how often the agent follows policies, completes tasks successfully, and respects boundaries.

## Why It Matters

Not all agents should have the same permissions. A new agent just starting out should probably have limited access until it proves itself trustworthy. An established agent with a perfect track record might deserve elevated privileges.

Trust Scoring makes this automatic:

- **High trust (900-1000)** → Full access, cross-organization delegation
- **Medium trust (500-699)** → Standard permissions
- **Low trust (300-499)** → Restricted, closely monitored
- **Very low trust (0-299)** → Read-only or completely blocked

## How It Works

The Trust Score is like a reputation system, but with mathematical rigor:

1. **Starting Score** — New agents start at 500 (Standard tier)
2. **Positive Events** — Following policies, completing tasks, respecting boundaries
3. **Negative Events** — Policy violations, errors, suspicious behavior
4. **Time Decay** — Very old positive events count less than recent ones

Every action the agent takes feeds into this scoring system, creating a dynamic, accurate picture of that agent's trustworthiness.

## Real-World Analogy

Think of it like a **driver's license points system**:

- New drivers start with a clean record
- Speeding tickets lower your score
- Accident-free driving maintains or improves it
- Too many points and you lose your license

Trust Scoring works the same way for AI agents.

## Key Benefits

1. **Dynamic Permissions** — Trust levels adjust based on behavior
2. **Risk-Based Access** — Higher-risk actions require higher trust scores
3. **Automatic Adjustment** — No manual intervention needed
4. **Clear Metrics** — Easy to understand, explain, and audit

## The Bottom Line

Trust Scoring transforms AI governance from a static, one-size-fits-all model into a **dynamic, behavior-based system** where permissions match proven trustworthiness.
