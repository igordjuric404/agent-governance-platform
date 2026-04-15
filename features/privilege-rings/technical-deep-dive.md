# Privilege Rings — Technical Deep Dive

## Architecture Overview

Privilege Rings implement **hierarchical capability enforcement** modeled after hardware privilege rings. Each ring represents a distinct capability boundary, with agents operating at a specific ring level enforced by the hypervisor.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Privilege Ring Model                           │
│                                                                  │
│  Ring 0 (Admin)  ─────────────────────────────────────────     │
│  │  • Full tool access                                          │
│  │  • All system calls                                           │
│  │  • Network unrestricted                                       │
│  │  • File system: unrestricted                                  │
│  ├─────────────────────────────────────────────────────────┤    │
│  Ring 1 (Standard) ────────────────────────────────────────     │
│  │  • Scoped tool access                                        │
│  │  • Most system calls                                          │
│  │  • Network: scoped to approved endpoints                      │
│  │  • File system: approved directories only                      │
│  ├─────────────────────────────────────────────────────────┤    │
│  Ring 2 (Restricted) ──────────────────────────────────────     │
│  │  • Read-only + limited writes                                 │
│  │  • Restricted system calls                                     │
│  │  • Network: allowlist only                                    │
│  │  • File system: working directory only                         │
│  ├─────────────────────────────────────────────────────────┤    │
│  Ring 3 (Sandboxed) ──────────────────────────────────────      │
│  │  • No external access                                         │
│  │  • Minimal system calls                                        │
│  │  • Network: none                                              │
│  │  • File system: temp directory only                            │
└─────────────────────────────────────────────────────────────────┘
```

## Ring Definitions

```python
class PrivilegeRing(Enum):
    """Four-tier privilege ring model."""
    
    RING_0_ADMIN = 0
    RING_1_STANDARD = 1
    RING_2_RESTRICTED = 2
    RING_3_SANDBOXED = 3

@dataclass
class RingConfiguration:
    """Configuration for each privilege ring."""
    
    ring: PrivilegeRing
    display_name: str
    capability_set: FrozenSet[str]
    denied_capabilities: FrozenSet[str]
    resource_limits: ResourceLimits
    network_policy: NetworkPolicy
    filesystem_scope: FilesystemScope
    min_trust_score: int
    requires_approval: bool
    audit_level: AuditLevel
```

## Ring Capability Mappings

```python
RING_CAPABILITIES = {
    PrivilegeRing.RING_0_ADMIN: {
        # All capabilities allowed
        'file.*',           # All file operations
        'network.*',        # All network operations
        'system.*',         # All system operations
        'agent.*',          # All agent operations
        'data.*',           # All data operations
        'code.execute',     # Code execution
        'code.elevated',    # Elevated code execution
    },
    PrivilegeRing.RING_1_STANDARD: {
        # Scoped standard operations
        'file.read',
        'file.write:/approved/*',
        'network.request:/approved/*',
        'network.request:/api/*',
        'system.command:read_only',
        'agent.message',
        'data.read',
        'data.search',
        'code.execute',
    },
    PrivilegeRing.RING_2_RESTRICTED: {
        # Read-only with limited writes
        'file.read',
        'file.write:/tmp/*',
        'network.request:/allowlist/*',
        'system.command:status_only',
        'agent.message:read_only',
        'data.read',
    },
    PrivilegeRing.RING_3_SANDBOXED: {
        # Minimal access
        'file.read:/readonly/*',
        'file.write:/tmp/sandbox/*',
        'network.request:localhost_only',
        'system.command:none',
        'agent.message:none',
        'data.read:/public/*',
    },
}
```

## Ring Classifier

```python
class RingClassifier:
    """Determines appropriate ring for an agent."""
    
    def classify(self, agent: AgentIdentity, 
                task: Task) -> PrivilegeRing:
        
        # Check trust score
        trust = self.trust_store.get_score(agent.id)
        
        # Check agent type/capabilities
        if agent.is_orchestrator:
            return PrivilegeRing.RING_0_ADMIN
        
        if trust.score >= 700 and agent.is_production_ready:
            return PrivilegeRing.RING_1_STANDARD
        
        if trust.score >= 300:
            return PrivilegeRing.RING_2_RESTRICTED
        
        return PrivilegeRing.RING_3_SANDBOXED
    
    def get_ring_transition(
        self, 
        agent_id: str,
        from_ring: PrivilegeRing,
        to_ring: PrivilegeRing,
        reason: str
    ) -> RingTransition:
        
        # Validate transition is allowed
        if not self._is_valid_transition(from_ring, to_ring):
            raise InvalidRingTransition(
                f"Cannot transition from {from_ring} to {to_ring}"
            )
        
        # Create transition record
        transition = RingTransition(
            agent_id=agent_id,
            from_ring=from_ring,
            to_ring=to_ring,
            reason=reason,
            approved_by=self._get_approver(from_ring, to_ring),
            timestamp=datetime.utcnow()
        )
        
        # Emit event
        self.event_bus.publish(AgentRingChanged(transition))
        
        return transition
```

## Ring Enforcer

```python
class RingEnforcer:
    """Enforces ring boundaries on agent operations."""
    
    def __init__(self, ring_config: Dict[PrivilegeRing, RingConfiguration]):
        self.config = ring_config
    
    def check_operation(
        self,
        agent_ring: PrivilegeRing,
        operation: Operation
    ) -> EnforcementDecision:
        
        ring_cfg = self.config[agent_ring]
        
        # Check capability
        if operation.required_capability not in ring_cfg.capability_set:
            if operation.required_capability not in ring_cfg.denied_capabilities:
                # Not in allowed set - apply default deny
                return EnforcementDecision(
                    allowed=False,
                    reason=f"Capability {operation.required_capability} not in ring {agent_ring}",
                    ring=agent_ring,
                    required_ring=self._find_minimum_ring(operation)
                )
        
        # Check resource constraints
        if not self._check_resource_constraints(ring_cfg, operation):
            return EnforcementDecision(
                allowed=False,
                reason="Resource constraints not met",
                ring=agent_ring
            )
        
        # Check filesystem scope
        if not self._check_filesystem_scope(ring_cfg, operation):
            return EnforcementDecision(
                allowed=False,
                reason=f"Filesystem access {operation.path} outside ring scope",
                ring=agent_ring,
                allowed_scope=ring_cfg.filesystem_scope
            )
        
        # Check network policy
        if not self._check_network_policy(ring_cfg, operation):
            return EnforcementDecision(
                allowed=False,
                reason=f"Network access {operation.endpoint} not allowed in ring",
                ring=agent_ring,
                allowed_policy=ring_cfg.network_policy
            )
        
        return EnforcementDecision(
            allowed=True,
            ring=agent_ring
        )
    
    def _find_minimum_ring(self, operation: Operation) -> PrivilegeRing:
        """Find minimum ring that can perform this operation."""
        for ring in [PrivilegeRing.RING_0_ADMIN, 
                     PrivilegeRing.RING_1_STANDARD,
                     PrivilegeRing.RING_2_RESTRICTED,
                     PrivilegeRing.RING_3_SANDBOXED]:
            if operation.required_capability in self.config[ring].capability_set:
                return ring
        return PrivilegeRing.RING_3_SANDBOXED
```

## Ring Transition Rules

```python
RING_TRANSITION_RULES = {
    # (from_ring, to_ring): requires_approval_from
    (PrivilegeRing.RING_3_SANDBOXED, PrivilegeRing.RING_2_RESTRICTED): True,
    (PrivilegeRing.RING_2_RESTRICTED, PrivilegeRing.RING_1_STANDARD): True,
    (PrivilegeRing.RING_1_STANDARD, PrivilegeRing.RING_0_ADMIN): True,
    # Downgrades don't require approval
    (PrivilegeRing.RING_1_STANDARD, PrivilegeRing.RING_2_RESTRICTED): False,
    (PrivilegeRing.RING_2_RESTRICTED, PrivilegeRing.RING_3_SANDBOXED): False,
}

class RingElevationRequest:
    """Request for ring elevation."""
    
    def __init__(
        self,
        agent_id: str,
        current_ring: PrivilegeRing,
        requested_ring: PrivilegeRing,
        justification: str,
        duration: Optional[timedelta]
    ):
        self.agent_id = agent_id
        self.current_ring = current_ring
        self.requested_ring = requested_ring
        self.justification = justification
        self.duration = duration
        self.status = RequestStatus.PENDING
        self.decision: Optional[RingDecision] = None
```

## Integration with Trust Scoring

```python
class TrustAwareRingSelector:
    """Selects ring based on trust score."""
    
    TRUST_TO_RING_MAPPING = {
        # (min_score, max_score): ring
        (900, 1000): PrivilegeRing.RING_0_ADMIN,
        (700, 899): PrivilegeRing.RING_1_STANDARD,
        (300, 699): PrivilegeRing.RING_2_RESTRICTED,
        (0, 299): PrivilegeRing.RING_3_SANDBOXED,
    }
    
    def select_ring(self, trust_score: int) -> PrivilegeRing:
        for (min_score, max_score), ring in self.TRUST_TO_RING_MAPPING.items():
            if min_score <= trust_score <= max_score:
                return ring
        return PrivilegeRing.RING_3_SANDBOXED
```

## Audit Logging

```python
@dataclass
class RingAuditRecord:
    """Audit record for ring operations."""
    
    agent_id: str
    ring: PrivilegeRing
    operation: str
    resource: str
    allowed: bool
    reason: Optional[str]
    timestamp: datetime
    ring_transition: Optional[RingTransition]
```

## Hypervisor Integration

```python
class HypervisorRingManager:
    """Manages rings through hypervisor."""
    
    def __init__(self, hypervisor: Hypervisor):
        self.hypervisor = hypervisor
    
    def assign_agent_to_ring(
        self, 
        agent_id: str, 
        ring: PrivilegeRing
    ) -> None:
        """Assign agent to specific ring."""
        
        # Configure cgroup limits based on ring
        cgroup_config = self._get_cgroup_config(ring)
        self.hypervisor.apply_cgroup_limits(agent_id, cgroup_config)
        
        # Apply seccomp profile based on ring
        seccomp_profile = self._get_seccomp_profile(ring)
        self.hypervisor.apply_seccomp(agent_id, seccomp_profile)
        
        # Set filesystem namespace
        fs_scope = self._get_filesystem_scope(ring)
        self.hypervisor.apply_filesystem_scope(agent_id, fs_scope)
```

## Performance

| Ring Level | Enforcement Latency | Memory Overhead |
|------------|-------------------|-----------------|
| Ring 0 | ~0.01 ms | Minimal |
| Ring 1 | ~0.02 ms | Minimal |
| Ring 2 | ~0.03 ms | Low |
| Ring 3 | ~0.05 ms | Low |
