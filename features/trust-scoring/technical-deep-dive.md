# Trust Scoring — Technical Deep Dive

## Architecture Overview

Trust Scoring in Ophanix implements a **behavior-based reputation system** that continuously evaluates agent trustworthiness through multiple signal dimensions. The system operates as a closed-loop feedback mechanism, adjusting scores based on real-time behavioral data.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Trust Scoring Architecture                     │
│                                                                  │
│  ┌──────────────┐                                               │
│  │   Agent      │                                               │
│  │   Action     │                                               │
│  └──────┬───────┘                                               │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Trust Event Collector                         │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                              │                                    │
│         ┌────────────────────┼────────────────────┐              │
│         ▼                    ▼                    ▼              │
│  ┌────────────┐     ┌────────────┐     ┌────────────┐         │
│  │  Policy    │     │  Task     │     │ Anomaly    │         │
│  │  Events    │     │  Success  │     │ Detection  │         │
│  └─────┬──────┘     └─────┬──────┘     └─────┬──────┘         │
│        │                   │                   │                  │
│        └───────────────────┼───────────────────┘                  │
│                            ▼                                      │
│                   ┌───────────────┐                              │
│                   │ Trust Engine  │                              │
│                   └───────┬───────┘                              │
│                           │                                        │
│         ┌─────────────────┼─────────────────┐                    │
│         ▼                 ▼                 ▼                    │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐             │
│  │  Score     │   │   Tier     │   │   Audit    │             │
│  │  Update    │   │  Lookup    │   │   Trail    │             │
│  └────────────┘   └────────────┘   └────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

## Trust Score Components

```python
@dataclass
class TrustScore:
    """Complete trust score for an agent."""
    
    agent_id: str
    score: int                          # 0-1000
    tier: TrustTier
    components: TrustComponents
    history: List[TrustEvent]
    last_updated: datetime
    next_review: datetime

@dataclass
class TrustComponents:
    """Individual trust score components."""
    
    policy_compliance: float    # 0.0-1.0 (policy adherence rate)
    task_success_rate: float    # 0.0-1.0 (successful task completion)
    boundary_respect: float     # 0.0-1.0 (respect for restrictions)
    anomaly_score: float         # 0.0-1.0 (0=anomalous, 1=normal)
    temporal_decay: float        # 0.0-1.0 (recency weighting)

@dataclass 
class TrustEvent:
    """Individual event affecting trust score."""
    
    event_type: TrustEventType
    agent_id: str
    delta: int                      # Score change (+/-)
    trigger: str                    # What caused this
    context: Dict[str, Any]          # Additional context
    timestamp: datetime
    verified: bool                   # Whether verified by multiple sources
```

## Trust Tiers

```python
class TrustTier(Enum):
    """Trust tier classification."""
    
    VERIFIED_PARTNER = (900, 1000, "cryptographically_verified")
    TRUSTED = (700, 899, "established_track_record")
    STANDARD = (500, 699, "default_new_agent")
    PROBATIONARY = (300, 499, "limited_under_observation")
    UNTRUSTED = (0, 299, "restricted_or_blocked")

@dataclass
class TierPrivileges:
    """Privileges associated with each tier."""
    
    tier: TrustTier
    can_delegate: bool
    cross_org_access: bool
    resource_limits: Dict[str, Any]
    requires_approval: List[str]
    rate_limit_multiplier: float
```

## Trust Score Algorithm

### Base Calculation

```python
class TrustScoreCalculator:
    """Calculates trust scores based on behavioral signals."""
    
    def calculate(self, components: TrustComponents) -> int:
        # Weighted average of components
        weights = {
            'policy_compliance': 0.35,
            'task_success_rate': 0.25,
            'boundary_respect': 0.20,
            'anomaly_score': 0.15,
            'temporal_decay': 0.05
        }
        
        raw_score = sum(
            getattr(components, k) * v 
            for k, v in weights.items()
        )
        
        # Scale to 0-1000
        return int(raw_score * 1000)
```

### Event-Based Updates

```python
class TrustScoreUpdater:
    """Updates trust scores based on events."""
    
    # Event multipliers
    EVENT_DELTAS = {
        TrustEventType.POLICY_VIOLATION: -50,
        TrustEventType.POLICY_VIOLATION_DENIED: -10,
        TrustEventType.TASK_SUCCESS: +5,
        TrustEventType.TASK_FAILURE: -15,
        TrustEventType.BOUNDARY_WARNING: -20,
        TrustEventType.ANOMALY_DETECTED: -30,
        TrustEventType.REVOCATION_RESTORED: +100,
        TrustEventType.IDENTITY_VERIFIED: +25,
    }
    
    def update(self, current_score: int, event: TrustEvent) -> int:
        delta = self.EVENT_DELTAS.get(event.event_type, 0)
        
        # Apply tier-based caps
        max_increase = self._get_max_increase(current_score)
        max_decrease = self._get_max_decrease(current_score)
        
        delta = max(min(delta, max_increase), -max_decrease)
        
        new_score = current_score + delta
        return max(0, min(1000, new_score))
```

### Temporal Decay

```python
class TemporalDecayCalculator:
    """Applies time-based decay to trust scores."""
    
    # Score half-life: how long until positive events count 50%
    HALF_LIFE_DAYS = 90
    
    def calculate_decay(self, last_update: datetime) -> float:
        age_days = (datetime.utcnow() - last_update).days
        
        if age_days <= 0:
            return 1.0
        
        # Exponential decay: 0.5^(age/half_life)
        decay = 0.5 ** (age_days / self.HALF_LIFE_DAYS)
        return max(0.5, decay)  # Never below 50%
```

## Trust Boundary Detection

```python
class TrustBoundaryMonitor:
    """Monitors for trust boundary violations."""
    
    def check_boundary(self, agent_id: str, score: int, 
                       requested_tier: TrustTier) -> BoundaryCheckResult:
        
        required_score = self._get_minimum_score(requested_tier)
        
        if score >= required_score:
            return BoundaryCheckResult(
                allowed=True,
                current_score=score,
                required_score=required_score
            )
        
        # Log potential privilege escalation attempt
        self.audit_logger.log_privilege_attempt(
            agent_id=agent_id,
            requested_tier=requested_tier,
            current_score=score,
            denied=True
        )
        
        return BoundaryCheckResult(
            allowed=False,
            current_score=score,
            required_score=required_score,
            gap=required_score - score
        )
```

## Integration with Policy Engine

```python
class TrustAwarePolicyEvaluator:
    """Policy evaluator that considers trust scores."""
    
    def evaluate(self, context: PolicyContext) -> PolicyDecision:
        agent_trust = self.trust_store.get_score(context.agent_id)
        
        # Add trust context to evaluation
        context.trust_score = agent_trust.score
        context.trust_tier = agent_trust.tier
        
        # Check if action requires elevated trust
        required_trust = self._get_required_trust(context.action)
        
        if agent_trust.score < required_trust:
            return PolicyDecision(
                allowed=False,
                reason=f"Trust score {agent_trust.score} below required {required_trust}",
                requires_escalation=True
            )
        
        # Standard policy evaluation
        return self.base_evaluator.evaluate(context)
```

## Audit Trail

```python
@dataclass
class TrustAuditRecord:
    """Immutable audit record for trust score changes."""
    
    agent_id: str
    previous_score: int
    new_score: int
    event_type: str
    event_details: Dict[str, Any]
    calculated_by: str              # Algorithm version
    signature: bytes               # For integrity verification
    
class TrustAuditLog:
    """Append-only audit log for trust score changes."""
    
    def append(self, record: TrustAuditRecord) -> None:
        # Hash chain for tamper evidence
        record.previous_hash = self._get_last_hash()
        record.hash = self._calculate_hash(record)
        
        self.storage.append(record)
    
    def verify(self, agent_id: str) -> bool:
        # Verify hash chain integrity
        pass
```

## Performance Characteristics

| Operation | Latency |
|-----------|---------|
| Score lookup | ~0.1 ms |
| Score update (event) | ~1 ms |
| Full recalculation | ~5 ms |
| Tier change event | ~2 ms |
| Audit record creation | ~0.5 ms |

## Storage Backend

```python
class TrustStore(ABC):
    """Abstract trust score storage."""
    
    @abstractmethod
    def get_score(self, agent_id: str) -> TrustScore:
        pass
    
    @abstractmethod
    def update_score(self, agent_id: str, event: TrustEvent) -> TrustScore:
        pass
    
    @abstractmethod
    def get_history(self, agent_id: str, 
                    since: datetime) -> List[TrustEvent]:
        pass
```

Implementations: Redis, PostgreSQL, In-memory for testing
