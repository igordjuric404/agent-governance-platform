# Least-Privilege Capabilities — Non-Technical Overview

## What Are Least-Privilege Capabilities?

The **principle of least privilege** is simple: give someone only the minimum access they need to do their job — nothing more.

For AI agents, this means: an agent that only needs to read documents shouldn't be able to delete them. An agent that searches the web shouldn't automatically have access to your financial systems.

**Least-Privilege Capabilities** is the feature that enforces this principle. It defines exactly what each agent **can** and **cannot** do, based on their role.

## Why It Matters

Without least-privilege enforcement:

- A web search agent could accidentally delete important files
- A data analysis agent might access customer records it shouldn't see
- A compromised agent could cause damage far beyond its intended scope

With least-privilege enforcement, even if something goes wrong, the damage is **contained** because the agent simply can't do things outside its defined scope.

## How It Works

When you create an agent, you define its **capabilities** — a list of specific things it's allowed to do:

```yaml
agent: customer-support-bot
capabilities:
  - read_knowledge_base
  - search_products
  - create_support_ticket
  - send_email
denied:
  - delete_data
  - access_financial_systems
  - modify_pricing
```

The system then enforces these capabilities every time the agent tries to take an action.

## Real-World Analogy

Think of it like a **company credit card**:

- A marketing employee: Can only charge to marketing budgets
- A CFO: Can charge to any business expense
- An intern: May have a card with very low limits or no card at all

Each person has exactly what they need for their role — nothing more.

## Key Benefits

1. **Damage Containment** — Problems are isolated to the agent's allowed scope
2. **Reduced Attack Surface** — Compromised agents have limited impact
3. **Compliance** — Easy to demonstrate "need to know" compliance
4. **Simplicity** — Clear expectations for what each agent should do

## The Bottom Line

Least-Privilege Capabilities ensures that every AI agent operates with **exactly the permissions it needs** — and nothing more. It's a fundamental security principle that protects your organization from both accidental errors and intentional misuse.
