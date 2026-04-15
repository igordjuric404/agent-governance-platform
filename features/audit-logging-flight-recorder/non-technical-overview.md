# Audit Logging & Flight Recorder — Non-Technical Overview

## What Is Audit Logging?

Every action an AI agent takes should leave a trace — a record of what happened, when, and why. This is called **audit logging**.

**Flight Recorder** takes this further, providing a complete, tamper-proof record of everything an agent has done — like the black box flight recorder on an airplane that investigators use to understand exactly what happened before an incident.

## Why It Matters

When something goes wrong with an AI agent, you need to answer critical questions:

- **What happened?** What actions did the agent take?
- **When did it happen?** Exact timing of each action?
- **Why did it happen?** What triggered the action?
- **Who is responsible?** Which agent took the action?
- **What was the outcome?** Did it succeed or fail?

Without good audit logging, these questions are impossible to answer. With it, you have a complete picture.

## How It Works

Every agent action generates an audit record:

```
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "agent_id": "customer-support-bot-001",
  "action": "send_email",
  "parameters": {
    "recipient": "customer@example.com",
    "subject": "Order Status Update"
  },
  "decision": "ALLOWED",
  "policy_evaluated": "customer-communication-policy",
  "trust_score": 750,
  "outcome": "SUCCESS"
}
```

This happens for EVERY action, creating an unbroken chain of evidence.

## Real-World Analogy

Think of audit logging like **security cameras in a bank**:

- Cameras record everything
- You don't watch every frame, but when something happens, you have evidence
- The recordings are tamper-proof
- They show exactly what happened, when

## Key Benefits

1. **Accountability** — Know exactly what each agent did
2. **Incident Investigation** — Reconstruct events after problems
3. **Compliance** — Meet regulatory requirements for audit trails
4. **Debugging** — Understand agent behavior when things go wrong
5. **Tamper-Proof** — Records cannot be altered or deleted

## The Bottom Line

Audit Logging and Flight Recorder provide complete visibility into AI agent operations. They turn "what happened?" from a mystery into a documented fact — essential for security, compliance, and continuous improvement.
