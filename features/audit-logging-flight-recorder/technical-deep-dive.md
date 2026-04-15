# Audit Logging & Flight Recorder — Technical Deep Dive

## Architecture Overview

Audit Logging and Flight Recorder implement an **immutable, append-only audit trail** for all AI agent operations. The system uses hash chaining to ensure tamper-evidence, similar to blockchain technology, while providing high-performance real-time querying.

```
┌─────────────────────────────────────────────────────────────────┐
│                   Audit Logging Architecture                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Audit Event Flow                         │  │
│  │                                                           │  │
│  │  Agent Action ──► Event Capture ──► Hash Chain           │  │
│  │                       │                  │                  │  │
│  │                       ▼                  ▼                  │  │
│  │              ┌──────────────┐     ┌──────────────┐        │  │
│  │              │  Immediate  │     │   Write-     │        │  │
│  │              │   Logger    │     │   Ahead Log  │        │  │
│  │              └──────┬───────┘     └──────────────┘        │  │
│  │                     │                                      │  │
│  └─────────────────────┼──────────────────────────────────────┘  │
│                        │                                          │
│                        ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Immutable Storage                          │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │              Hash Chain Integrity                      │ │  │
│  │  │  Block N ──► Hash(Block N) ──► Hash(Block N+1)     │ │  │
│  │  │      │                          │                    │ │  │
│  │  │      └──────────────────────────┘                    │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Data Structures

```python
@dataclass
class AuditRecord:
    """Immutable audit record for agent actions."""
    
    record_id: str                    # UUID for this record
    sequence_number: int              # Monotonic sequence
    timestamp: datetime               # Precise timestamp (UTC)
    
    # Identity
    agent_id: str                    # Which agent acted
    agent_trust_score: int           # Trust score at action time
    
    # Action details
    action_type: ActionType          # What the agent did
    action_details: Dict[str, Any]   # Action parameters
    resource_accessed: Optional[str] # Resource if applicable
    
    # Decision context
    decision: Decision               # ALLOW, DENY, PARTIAL
    policy_evaluated: Optional[str]  # Policy that applied
    policy_rule_matched: Optional[str]  # Specific rule
    denial_reason: Optional[str]     # Why denied if applicable
    
    # Outcome
    outcome: Outcome                 # SUCCESS, FAILURE, TIMEOUT
    error_details: Optional[str]     # Error message if failure
    
    # Chain integrity
    previous_hash: str               # Hash of previous record
    current_hash: str                # Hash of this record
    signature: bytes                 # Signature for non-repudiation

@dataclass
class FlightRecorderSnapshot:
    """Complete state capture at a point in time."""
    
    snapshot_id: str
    timestamp: datetime
    agent_id: str
    
    # Execution state
    current_action: Optional[str]
    action_history: List[AuditRecord]
    memory_state: Dict[str, Any]
    context_stack: List[str]
    
    # Policy state
    active_policies: List[str]
    recent_decisions: List[Decision]
    
    # Hash chain
    chain_hash: str
    snapshot_hash: str

class ActionType(Enum):
    """Types of auditable actions."""
    
    TOOL_CALL = "tool.call"
    RESOURCE_ACCESS = "resource.access"
    AGENT_MESSAGE = "agent.message"
    POLICY_EVALUATION = "policy.evaluation"
    TRUST_SCORE_CHANGE = "trust.score.change"
    IDENTITY_VERIFICATION = "identity.verification"
    PRIVILEGE_ESCALATION = "privilege.escalation"
    KILL_SWITCH_TRIGGER = "kill.switch.trigger"
    COMPENSATION_TRIGGER = "compensation.trigger"
```

## Hash Chain Implementation

```python
class AuditHashChain:
    """Implements tamper-evident hash chain for audit records."""
    
    def __init__(self, hash_function: Callable = hashlib.sha256):
        self.hash_function = hash_function
        self.genesis_hash = self._compute_genesis_hash()
    
    def compute_record_hash(
        self,
        record: AuditRecord,
        previous_hash: str
    ) -> str:
        """Compute hash for an audit record."""
        
        # Create deterministic representation
        content = self._serialize_for_hash(record)
        
        # Include previous hash in computation
        hash_input = f"{content}:{previous_hash}"
        
        return self.hash_function(hash_input.encode()).hexdigest()
    
    def verify_chain(self, records: List[AuditRecord]) -> VerificationResult:
        """Verify the integrity of an audit chain."""
        
        if not records:
            return VerificationResult(valid=True)
        
        # Check first record connects to genesis
        if records[0].previous_hash != self.genesis_hash:
            return VerificationResult(
                valid=False,
                error="Chain does not connect to genesis"
            )
        
        # Verify each record's hash
        for i in range(len(records) - 1):
            current = records[i]
            next_record = records[i + 1]
            
            # Recompute current hash
            expected_hash = self.compute_record_hash(
                current,
                current.previous_hash
            )
            
            if current.current_hash != expected_hash:
                return VerificationResult(
                    valid=False,
                    error=f"Record {current.record_id} hash mismatch",
                    corrupted_record=current.record_id
                )
            
            # Verify chain link
            if next_record.previous_hash != current.current_hash:
                return VerificationResult(
                    valid=False,
                    error=f"Chain broken between records {current.record_id} and {next_record.record_id}",
                    broken_at=current.record_id
                )
        
        return VerificationResult(valid=True)
    
    def _compute_genesis_hash(self) -> str:
        """Compute the genesis hash for the chain."""
        timestamp = datetime.utcnow().isoformat()
        return self.hash_function(f"GENESIS:{timestamp}".encode()).hexdigest()
```

## Flight Recorder

```python
class FlightRecorder:
    """Provides complete state capture for incident investigation."""
    
    def __init__(
        self,
        audit_store: AuditStore,
        state_capturer: StateCapturer
    ):
        self.audit_store = audit_store
        self.state_capturer = state_capturer
    
    async def capture_snapshot(
        self,
        agent_id: str,
        reason: str
    ) -> FlightRecorderSnapshot:
        """Capture complete agent state for incident investigation."""
        
        # Get recent audit history
        recent_records = await self.audit_store.get_recent(
            agent_id=agent_id,
            limit=1000
        )
        
        # Capture current state
        current_state = await self.state_capturer.capture(agent_id)
        
        # Create snapshot
        snapshot = FlightRecorderSnapshot(
            snapshot_id=uuid.uuid4().hex,
            timestamp=datetime.utcnow(),
            agent_id=agent_id,
            action_history=list(recent_records),
            memory_state=current_state.memory,
            context_stack=current_state.contexts,
            active_policies=current_state.policies,
            recent_decisions=[r.decision for r in recent_records[-100:]],
            chain_hash=self._compute_chain_hash(recent_records),
            snapshot_hash=self._compute_snapshot_hash(recent_records, current_state)
        )
        
        # Store snapshot
        await self.audit_store.store_snapshot(snapshot)
        
        return snapshot
    
    async def replay_to_point(
        self,
        agent_id: str,
        target_timestamp: datetime
    ) -> List[AuditRecord]:
        """Replay audit records up to a specific point in time."""
        
        records = await self.audit_store.get_before(
            agent_id=agent_id,
            timestamp=target_timestamp
        )
        
        return sorted(records, key=lambda r: r.sequence_number)
```

## Event Capture

```python
class AuditEventCapture:
    """Captures audit events in real-time."""
    
    def __init__(
        self,
        event_bus: EventBus,
        audit_writer: AuditWriter,
        credential_redactor: CredentialRedactor
    ):
        self.event_bus = event_bus
        self.audit_writer = audit_writer
        self.redactor = credential_redactor
        self.sequence = 0
        self.last_hash = None
    
    async def capture(self, event: AgentEvent) -> AuditRecord:
        """Capture an agent event as an audit record."""
        
        # Increment sequence
        self.sequence += 1
        
        # Redact sensitive data
        redacted_event = self.redactor.redact(event)
        
        # Create audit record
        record = AuditRecord(
            record_id=uuid.uuid4().hex,
            sequence_number=self.sequence,
            timestamp=datetime.utcnow(),
            agent_id=event.agent_id,
            agent_trust_score=event.trust_score,
            action_type=self._map_to_action_type(event),
            action_details=redacted_event.details,
            resource_accessed=event.resource,
            decision=event.decision,
            policy_evaluated=event.policy,
            policy_rule_matched=event.rule,
            denial_reason=event.denial_reason,
            outcome=event.outcome,
            error_details=event.error,
            previous_hash=self.last_hash or "",
            current_hash="",  # Will be computed
            signature=b""  # Will be signed
        )
        
        # Compute hash chain link
        record.current_hash = self.audit_writer.compute_record_hash(
            record,
            record.previous_hash
        )
        
        # Sign for non-repudiation
        record.signature = self._sign_record(record)
        
        # Update chain state
        self.last_hash = record.current_hash
        
        # Write to storage
        await self.audit_writer.write(record)
        
        return record
    
    def _sign_record(self, record: AuditRecord) -> bytes:
        """Sign record for non-repudiation."""
        # Sign with audit key (would use HSM in production)
        content = self._serialize_for_sign(record)
        return self.audit_signing_key.sign(content.encode())
```

## Query API

```python
class AuditQueryAPI:
    """Query interface for audit records."""
    
    async def query(
        self,
        filters: AuditQueryFilters,
        pagination: Pagination
    ) -> AuditQueryResult:
        """Query audit records with filters."""
        
        records = await self.audit_store.query(
            agent_ids=filters.agent_ids,
            action_types=filters.action_types,
            start_time=filters.start_time,
            end_time=filters.end_time,
            decisions=filters.decisions,
            outcomes=filters.outcomes,
            limit=pagination.limit,
            offset=pagination.offset
        )
        
        return AuditQueryResult(
            records=records,
            total=len(records),
            has_more=pagination.has_more
        )
    
    async def get_agent_timeline(
        self,
        agent_id: str,
        start_time: datetime,
        end_time: datetime
    ) -> List[TimelineEvent]:
        """Get chronological timeline of agent actions."""
        
        records = await self.audit_store.query(
            agent_ids=[agent_id],
            start_time=start_time,
            end_time=end_time
        )
        
        return self._build_timeline(records)
    
    async def get_incident_reconstruction(
        self,
        incident_id: str
    ) -> IncidentReconstruction:
        """Reconstruct complete incident from audit trail."""
        
        incident = await self.incident_store.get(incident_id)
        
        # Get all agents involved
        agent_ids = incident.affected_agents
        
        # Get records for all agents during incident window
        all_records = []
        for agent_id in agent_ids:
            records = await self.audit_store.query(
                agent_ids=[agent_id],
                start_time=incident.start_time,
                end_time=incident.end_time
            )
            all_records.extend(records)
        
        # Sort by timestamp
        all_records.sort(key=lambda r: r.timestamp)
        
        return IncidentReconstruction(
            incident=incident,
            records=all_records,
            verification=self._verify_chain(all_records)
        )
```

## Credential Redaction

```python
class CredentialRedactor:
    """Redacts sensitive data from audit records."""
    
    SENSITIVE_PATTERNS = [
        (r'password["\']?\s*[:=]\s*["\'][^"\']+["\']', '***REDACTED***'),
        (r'api[_-]?key["\']?\s*[:=]\s*["\'][^"\']+["\']', '***REDACTED***'),
        (r'token["\']?\s*[:=]\s*["\'][^"\']+["\']', '***REDACTED***'),
        (r'secret["\']?\s*[:=]\s*["\'][^"\']+["\']', '***REDACTED***'),
        (r'ssn["\']?\s*[:=]\s*["\'][^"\']+["\']', '***REDACTED***'),
        (r'\b\d{3}-\d{2}-\d{4}\b', '***SSN-REDACTED***'),
    ]
    
    def redact(self, event: AgentEvent) -> AgentEvent:
        """Remove sensitive data from event."""
        
        redacted_details = {}
        
        for key, value in event.details.items():
            if isinstance(value, str):
                redacted_value = self._redact_string(value)
            elif isinstance(value, dict):
                redacted_value = {k: self._redact_string(v) for k, v in value.items()}
            else:
                redacted_value = value
            
            redacted_details[key] = redacted_value
        
        event.details = redacted_details
        return event
    
    def _redact_string(self, text: str) -> str:
        """Apply all redaction patterns to string."""
        
        for pattern, replacement in self.SENSITIVE_PATTERNS:
            text = re.sub(pattern, replacement, text, flags=re.IGNORECASE)
        
        return text
```

## Performance Characteristics

| Metric | Value |
|--------|-------|
| Record write latency | ~2 ms |
| Chain verification | ~10 ms per 1000 records |
| Query by agent/time | ~50 ms |
| Snapshot creation | ~100 ms |
| Storage per record | ~2 KB |
