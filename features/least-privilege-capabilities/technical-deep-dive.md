# Least-Privilege Capabilities — Technical Deep Dive

## Architecture Overview

Least-Privilege Capabilities implements **capability-based access control** where agent permissions are defined as explicit, bounded sets of allowed actions. This differs from traditional RBAC by being more granular and explicitly enumerating what an agent CAN do rather than what it cannot.

```
┌─────────────────────────────────────────────────────────────────┐
│               Least-Privilege Capability Model                    │
│                                                                  │
│  Capability Registry                                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  agent_id: "customer-support-bot"                          │  │
│  │  capabilities: [read_kb, search_prod, create_ticket]     │  │
│  │  denied: [delete_data, modify_pricing, access_financial]  │  │
│  │  resource_constraints: {...}                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Capability Enforcement Point                   │  │
│  │                                                           │  │
│  │  Request: {agent, action, resource}                       │  │
│  │            │                                             │  │
│  │            ▼                                             │  │
│  │  ┌─────────────────┐     ┌─────────────────┐          │  │
│  │  │ Capability      │────►│ Resource        │          │  │
│  │  │ Resolution      │     │ Constraints     │          │  │
│  │  └─────────────────┘     └─────────────────┘          │  │
│  │            │                                             │  │
│  │            ▼                                             │  │
│  │  ┌─────────────────────────────────────────────────┐    │  │
│  │  │           Enforcement Decision                    │    │  │
│  │  │  ALLOW: Capability granted & constraints met      │    │  │
│  │  │  DENY:  Capability missing or constraint failed  │    │  │
│  │  └─────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Data Structures

```python
@dataclass
class AgentCapabilities:
    """Defines what an agent is allowed to do."""
    
    agent_id: str
    capabilities: FrozenSet[Capability]       # Explicitly allowed
    denied_capabilities: FrozenSet[Capability]  # Explicitly denied
    resource_constraints: Dict[str, ResourceConstraint]
    temporal_constraints: Optional[TemporalConstraints]
    conditions: Optional[List[CapabilityCondition]]  # Conditional grants
    
@dataclass
class Capability:
    """A single capability/action."""
    
    name: str                              # e.g., "file.read"
    category: CapabilityCategory            # File, Network, Data, etc.
    risk_level: RiskLevel                  # LOW, MEDIUM, HIGH, CRITICAL
    requires_approval: bool                # Needs human approval
    min_trust_tier: TrustTier             # Minimum trust required
    audit_level: AuditLevel               # NONE, BASIC, FULL

@dataclass 
class ResourceConstraint:
    """Constraint on resource access."""
    
    resource_pattern: str                  # e.g., "/data/customers/*"
    permission: ResourcePermission          # READ, WRITE, DELETE
    max_size_bytes: Optional[int]          # For data operations
    rate_limit: Optional[RateLimit]        # Operations per time window

@dataclass
class CapabilityCondition:
    """Conditional capability grant."""
    
    condition_type: ConditionType          # TIME, CONTEXT, STATE
    expression: str                        # e.g., "hour < 18"
    granted_capability: Capability
    expires_at: Optional[datetime]
```

## Capability Categories

```python
class CapabilityCategory(Enum):
    """Categories of capabilities."""
    
    # Data Operations
    DATA_READ = "data.read"
    DATA_WRITE = "data.write"
    DATA_DELETE = "data.delete"
    
    # File Operations
    FILE_READ = "file.read"
    FILE_WRITE = "file.write"
    FILE_DELETE = "file.delete"
    FILE_EXECUTE = "file.execute"
    
    # Network Operations
    NETWORK_REQUEST = "network.request"
    NETWORK_RECEIVE = "network.receive"
    WEB_SEARCH = "web.search"
    
    # Agent Operations
    AGENT_CREATE = "agent.create"
    AGENT_DELETE = "agent.delete"
    AGENT_DELEGATE = "agent.delegate"
    AGENT_MESSAGE = "agent.message"
    
    # System Operations
    SYSTEM_COMMAND = "system.command"
    CODE_EXECUTE = "code.execute"
    SHELL_EXECUTE = "shell.execute"
    
    # Communication
    EMAIL_SEND = "email.send"
    EMAIL_READ = "email.read"
    NOTIFICATION_SEND = "notification.send"
```

## Capability Resolution

```python
class CapabilityResolver:
    """Resolves whether an agent has a specific capability."""
    
    def __init__(self, registry: CapabilityRegistry):
        self.registry = registry
    
    def resolve(
        self, 
        agent_id: str, 
        capability: Capability,
        resource: Optional[str] = None
    ) -> CapabilityResolution:
        
        caps = self.registry.get_capabilities(agent_id)
        
        # Check explicit denial first (deny always wins)
        if capability in caps.denied_capabilities:
            return CapabilityResolution(
                allowed=False,
                reason="Explicitly denied",
                source="deny_list"
            )
        
        # Check if capability is in granted set
        if capability in caps.capabilities:
            # Validate resource constraints
            if resource:
                constraint = self._get_resource_constraint(caps, capability)
                if constraint and not self._check_constraint(constraint, resource):
                    return CapabilityResolution(
                        allowed=False,
                        reason=f"Resource constraint not met for {resource}",
                        source="constraint_check"
                    )
            
            # Check trust tier requirement
            if caps.agent_id in self._get_trust_requirements(capability):
                trust = self.trust_store.get_score(caps.agent_id)
                if trust.tier < capability.min_trust_tier:
                    return CapabilityResolution(
                        allowed=False,
                        reason=f"Trust tier {trust.tier} below required {capability.min_trust_tier}",
                        source="trust_check"
                    )
            
            return CapabilityResolution(
                allowed=True,
                reason="Capability granted",
                source="capability_list"
            )
        
        # Check conditional capabilities
        for condition in caps.conditions or []:
            if self._evaluate_condition(condition, capability):
                return CapabilityResolution(
                    allowed=True,
                    reason=f"Conditional capability: {condition.condition_type}",
                    source="conditional_grant"
                )
        
        # Default deny (principle of least privilege)
        return CapabilityResolution(
            allowed=False,
            reason="Capability not granted",
            source="default_deny"
        )
```

## Constraint Graph

```python
class ConstraintGraph:
    """Manages capability constraints as a directed graph."""
    
    def __init__(self):
        self.nodes: Dict[str, ConstraintNode] = {}
        self.edges: Dict[str, List[str]] = defaultdict(list)
    
    def add_capability(
        self, 
        capability: Capability,
        constraints: Dict[str, Any]
    ) -> None:
        node = ConstraintNode(
            capability=capability,
            constraints=constraints,
            depends_on=constraints.get('depends_on', [])
        )
        self.nodes[capability.name] = node
        
        for dep in node.depends_on:
            self.edges[dep].append(capability.name)
    
    def validate(self, requested: Capability) -> ValidationResult:
        """Validate that all dependencies are satisfied."""
        if requested not in self.nodes:
            return ValidationResult(valid=True)  # No constraints
        
        node = self.nodes[requested]
        satisfied = self._check_constraints(node.constraints)
        
        if not satisfied:
            return ValidationResult(
                valid=False,
                reason=f"Constraints not satisfied for {requested.name}",
                failed_constraints=node.constraints
            )
        
        return ValidationResult(valid=True)
```

## Integration with Policy Engine

```python
class LeastPrivilegePolicyHook:
    """Policy engine integration for least-privilege enforcement."""
    
    def evaluate(self, context: PolicyContext) -> PolicyDecision:
        action = context.action
        agent_id = context.agent_id
        
        # Get the required capability for this action
        required_capability = self._map_action_to_capability(action)
        
        # Resolve capability
        resolution = self.capability_resolver.resolve(
            agent_id=agent_id,
            capability=required_capability,
            resource=context.resource
        )
        
        if not resolution.allowed:
            # Log capability denial
            self.audit_logger.log_capability_denial(
                agent_id=agent_id,
                capability=required_capability,
                reason=resolution.reason
            )
            
            return PolicyDecision(
                allowed=False,
                reason=f"Capability denied: {resolution.reason}",
                code="CAPABILITY_NOT_GRANTED"
            )
        
        return PolicyDecision(allowed=True)
```

## Capability Inheritance

```python
class CapabilityInheritance:
    """Handles capability inheritance through delegation."""
    
    def delegate_capabilities(
        self,
        from_agent: str,
        to_agent: str,
        capabilities: List[Capability],
        constraints: DelegationConstraints
    ) -> DelegatedCapabilities:
        
        # Inherit base capabilities from delegator
        inherited = self.registry.get_capabilities(from_agent).capabilities
        
        # Intersect with explicitly delegated
        allowed = frozenset(capabilities) & inherited
        
        # Apply additional delegation constraints
        constrained = self._apply_constraints(allowed, constraints)
        
        return DelegatedCapabilities(
            agent_id=to_agent,
            capabilities=constrained,
            delegated_by=from_agent,
            expires_at=constraints.expires_at,
            constraints=constraints
        )
```

## YAML Definition Example

```yaml
capabilities:
  customer-support-bot:
    description: "Handles customer inquiries and ticket creation"
    
    capabilities:
      - data.read:knowledge_base
      - data.search:products
      - ticket.create
      - ticket.read
      - notification.send:email
      - notification.send:slack
    
    denied:
      - data.delete
      - data.export
      - financial.access
      - system.command
    
    resource_constraints:
      data.read:
        allowed_paths:
          - /knowledge_base/*
          - /products/*
        max_results: 100
      
      notification.send:
        rate_limit: 100/hour
        requires_template: true
    
    conditions:
      - type: time
        expression: "hour >= 9 AND hour <= 17"
        grants:
          - ticket.create
```

## Security Considerations

1. **Deny Wins** — Explicit denials always override grants
2. **Default Deny** — Unlisted capabilities are denied by default
3. **Capability Auditing** — All capability checks logged
4. **Constraint Validation** — Resources validated before access
5. **Temporal Constraints** — Time-based restrictions enforced

## Performance

| Operation | Latency |
|-----------|---------|
| Capability lookup | ~0.05 ms |
| Constraint validation | ~0.1 ms |
| Full resolution | ~0.2 ms |
| Batch capability check | ~0.5 ms |
