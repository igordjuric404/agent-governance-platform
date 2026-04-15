# Policy Engine — Technical Deep Dive

## Architecture Overview

The Policy Engine is the central enforcement point of the Ophanix governance platform. It implements a **deterministic policy evaluation model** where every agent action is evaluated against a set of rules before execution is permitted.

```
┌─────────────────────────────────────────────────────────────┐
│                    Policy Evaluation Flow                     │
│                                                              │
│  Action Request ─► Context Builder ─► Rule Evaluator        │
│                          │                    │              │
│                          ▼                    ▼              │
│                   ┌────────────┐       ┌─────────────┐      │
│                   │  Policy    │       │   Conflict  │      │
│                   │  Loading   │       │  Resolution │      │
│                   └────────────┘       └─────────────┘      │
│                                               │              │
│                                               ▼              │
│                                        ┌─────────────┐      │
│                                        │  Decision   │      │
│                                        │  (Allow/    │      │
│                                        │   Deny)     │      │
│                                        └─────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### Policy Schema (`agent_os/policies/schema.py`)

```python
@dataclass
class PolicyDocument:
    name: str
    version: str
    description: Optional[str]
    defaults: PolicyDefaults
    rules: List[PolicyRule]

@dataclass
class PolicyRule:
    name: str
    condition: PolicyCondition
    action: PolicyAction  # ALLOW or DENY
    priority: int
    message: Optional[str]
    metadata: Optional[Dict[str, Any]]
```

### Condition Operators

| Operator | Description |
|----------|-------------|
| `EQ` | Exact equality match |
| `NEQ` | Not equal |
| `IN` | Value in list |
| `NOT_IN` | Value not in list |
| `MATCHES` | Regex pattern match |
| `GT`, `GTE`, `LT`, `LTE` | Numeric comparisons |
| `CONTAINS`, `NOT_CONTAINS` | Collection membership |

### Evaluation Backend (`agent_os/policies/backends.py`)

The engine supports multiple policy backends:

1. **YAML Backend** — Native YAML policy definition
2. **OPA/Rego Backend** — Open Policy Agent integration
3. **Cedar Backend** — Amazon Verified Permissions integration

```python
class PolicyEvaluator:
    def __init__(self, policies: List[PolicyDocument]):
        self.policies = policies
        self.backend = YAMLBackend()  # or OPABackend(), CedarBackend()
    
    def evaluate(self, context: Dict[str, Any]) -> PolicyDecision:
        # Build evaluation context
        eval_context = self._build_context(context)
        
        # Find matching rules by priority
        matching_rules = self._find_matching_rules(eval_context)
        
        # Apply conflict resolution
        decision = self._resolve_conflicts(matching_rules)
        
        # Return decision with full audit trail
        return PolicyDecision(
            allowed=decision.action == PolicyAction.ALLOW,
            reason=decision.message,
            rule=decision.name,
            timestamp=datetime.utcnow(),
            context_id=self._generate_context_id(context)
        )
```

## Conflict Resolution Strategy

When multiple rules match the same context, the engine uses a **priority-based conflict resolution** strategy:

1. **Highest Priority Wins** — Rule with highest priority value takes precedence
2. **Deny Overrides Allow** — At equal priority, DENY takes precedence
3. **First Match Wins** — Within same priority/action, first defined rule wins

## Performance Characteristics

| Metric | Value |
|--------|-------|
| Single rule evaluation | 0.012 ms |
| 100 rules evaluation | 0.029 ms |
| Full policy enforcement | 0.091 ms |
| Throughput | 72K ops/sec (1 rule), 31K ops/sec (100 rules) |

## Integration Points

### Framework Integration

The Policy Engine integrates with major agent frameworks via middleware:

- **LangChain** — `agent_os.integrations.langchain`
- **AutoGen** — `agent_os.integrations.autogen`
- **CrewAI** — `agent_os.integrations.crewai`
- **OpenAI Agents SDK** — `agent_os.integrations.openai_agents`

### Pre-Execution Hook

```python
@agent_framework_hook
def evaluate_action(action: AgentAction) -> PolicyDecision:
    context = {
        "agent_id": action.agent_id,
        "tool_name": action.tool,
        "parameters": action.params,
        "timestamp": datetime.utcnow().isoformat(),
    }
    return policy_engine.evaluate(context)
```

## YAML Policy Example

```yaml
name: enterprise-safety-policy
version: "1.0"
defaults:
  action: ALLOW

rules:
  - name: block-destructive-tools
    priority: 100
    condition:
      field: tool_name
      operator: IN
      value: ["delete_file", "execute_code", "shell_exec"]
    action: DENY
    message: "Tool blocked by security policy"

  - name: block-sensitive-data-exfiltration
    priority: 90
    condition:
      field: output_text
      operator: MATCHES
      value: "\\b\\d{3}-\\d{2}-\\d{4}\\b"  # SSN pattern
    action: DENY
    message: "Sensitive data pattern detected"

  - name: restrict-file-operations
    priority: 80
    condition:
      field: tool_name
      operator: EQ
      value: "write_file"
    condition:
      field: parameters.path
      operator: NOT_IN
      value: ["/allowed/path1", "/allowed/path2"]
    action: DENY
    message: "File write to non-allowed location"
```

## Security Considerations

1. **Immutable Audit Trail** — Every evaluation is logged with hash chaining
2. **Credential Redaction** — Sensitive values are redacted from logs
3. **Input Validation** — All context values are validated before evaluation
4. **Sandbox Execution** — Policy evaluation runs in isolated context

## Testing Strategy

- **Unit Tests** — Individual rule evaluation
- **Integration Tests** — Framework hook integration
- **Fuzz Testing** — `fuzz_condition_eval.py` for edge cases
- **Benchmark Tests** — Performance regression detection
