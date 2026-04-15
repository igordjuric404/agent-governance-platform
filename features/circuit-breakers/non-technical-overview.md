# Circuit Breakers — Non-Technical Overview

## What Is a Circuit Breaker?

If your home's electrical system has a circuit breaker, you already understand this concept. When too many devices draw power, the circuit trips and cuts off electricity — preventing a fire and protecting your appliances.

**Circuit Breakers for AI agents** work the same way. When an agent starts having too many failures or errors, the circuit breaker "trips" and temporarily stops the agent from taking further actions. This prevents cascading failures from bringing down your entire system.

## Why It's Needed

Imagine you have three AI agents working together on a task:

1. **Agent A** calls **Agent B** to get data
2. **Agent B** calls **Agent C** to process the data
3. **Agent C** starts failing

Without circuit breakers:
- Agent B keeps trying to call Agent C
- Agent B starts timing out
- Agent A is now waiting for Agent B
- Eventually, everything is stuck waiting for Agent C

This is called a **cascading failure** — one component's failure spreads to bring down the entire system.

With circuit breakers:
- Agent C starts failing
- After a threshold, the circuit breaker "trips"
- Agent B immediately gets an error saying "C is unavailable"
- Agent B can try an alternative or report the problem
- Agent A continues working

## How It Works

Circuit breakers have three states:

### Closed (Normal)
Everything is working. Requests pass through normally.
- Failures are counted
- If failures exceed a threshold, → Open

### Open (Tripped)
The protected agent is failing.
- Requests immediately fail without calling the agent
- After a timeout, → Half-Open

### Half-Open (Testing)
Testing if the agent has recovered.
- Limited requests pass through
- If they succeed, → Closed
- If they fail, → Open

## Real-World Analogy

Think of circuit breakers like **traffic lights at an intersection**:

- **Green (Closed):** Traffic flows normally
- **Red (Open):** Traffic stopped, find another route
- **Yellow (Half-Open):** Proceed with caution, testing

## Key Benefits

1. **Prevent Cascading Failures** — Stop failures from spreading
2. **Fast Failures** — Don't wait for timeouts
3. **Graceful Degradation** — System keeps working, just without the failing part
4. **Auto-Recovery** — Automatically tests if the problem is fixed

## The Bottom Line

Circuit Breakers ensure that when one AI agent fails, it doesn't bring down your entire system. They provide the resilience pattern that lets distributed AI systems survive component failures.
