# Kill Switch — Technical Deep Dive

## Architecture Overview

The Kill Switch implements **immediate agent termination** through multiple enforcement mechanisms. It's designed as a failsafe that operates independently of the agent's execution context, ensuring termination even if the agent is unresponsive or malicious.

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kill Switch Architecture                      │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Kill Switch Manager                      │  │
│  │                                                           │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │   Trigger    │  │  Auth &      │  │  Enforcement │  │  │
│  │  │   Monitor    │──│  Validation  │──│   Engine     │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
│  │         │                │                  │             │  │
│  │         └────────────────┼──────────────────┘             │  │
│  │                          ▼                                  │  │
│  │              ┌──────────────────┐                          │  │
│  │              │   Agent Registry │                          │  │
│  │              └──────────────────┘                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐            │
│         ▼                    ▼                    ▼            │
│  ┌────────────┐     ┌────────────┐     ┌────────────┐         │
│  │   Signal   │     │  Process   │     │  Network   │         │
│  │   SIGKILL  │     │   Tree     │     │  Blackhole │         │
│  └────────────┘     └────────────┘     └────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

```python
class KillSwitchManager:
    """Central manager for agent termination."""
    
    def __init__(
        self,
        agent_registry: AgentRegistry,
        signal_dispatcher: SignalDispatcher,
        process_killer: ProcessKiller,
        network_isolator: NetworkIsolator,
        audit_logger: AuditLogger
    ):
        self.registry = agent_registry
        self.signal_dispatcher = signal_dispatcher
        self.process_killer = process_killer
        self.network_isolator = network_isolator
        self.audit_logger = audit_logger
    
    async def terminate(
        self,
        agent_id: str,
        reason: KillSwitchReason,
        triggered_by: str,
        force: bool = True
    ) -> TerminationResult:
        """Immediately terminate an agent and all its processes."""
        
        # Validate authorization
        if not self._authorize_termination(triggered_by, agent_id):
            raise UnauthorizedTerminationAttempt(
                agent_id=agent_id,
                attempted_by=triggered_by
            )
        
        # Log initiation
        termination_id = uuid.uuid4().hex
        self.audit_logger.log_kill_switch_triggered(
            termination_id=termination_id,
            agent_id=agent_id,
            reason=reason,
            triggered_by=triggered_by,
            timestamp=datetime.utcnow()
        )
        
        # Get all processes for this agent
        processes = self.registry.get_agent_processes(agent_id)
        
        # Create termination context
        context = TerminationContext(
            termination_id=termination_id,
            agent_id=agent_id,
            processes=processes,
            reason=reason,
            force=force,
            started_at=datetime.utcnow()
        )
        
        # Execute termination
        try:
            # Step 1: Network isolation (prevent further communication)
            await self.network_isolator.isolate(agent_id)
            
            # Step 2: Send termination signal to all processes
            await self.process_killer.terminate_tree(
                processes, 
                signal=signal.SIGKILL if force else signal.SIGTERM
            )
            
            # Step 3: Cleanup resources
            await self._cleanup_resources(agent_id)
            
            # Step 4: Update registry
            self.registry.mark_terminated(agent_id, termination_id)
            
            result = TerminationResult(
                success=True,
                termination_id=termination_id,
                agent_id=agent_id,
                processes_terminated=len(processes),
                duration_ms=(datetime.utcnow() - context.started_at).total_seconds() * 1000
            )
            
            # Log completion
            self.audit_logger.log_kill_switch_completed(result)
            
            return result
            
        except Exception as e:
            # Log failure
            self.audit_logger.log_kill_switch_failed(
                termination_id=termination_id,
                error=str(e)
            )
            
            return TerminationResult(
                success=False,
                termination_id=termination_id,
                agent_id=agent_id,
                error=str(e)
            )
```

## Termination Triggers

```python
@dataclass
class KillSwitchReason:
    """Reason for kill switch activation."""
    
    category: KillSwitchCategory
    detail: str
    evidence: Dict[str, Any]
    severity: Severity

class KillSwitchCategory(Enum):
    """Categories of kill switch triggers."""
    
    MANUAL = "manual"                    # Human-initiated
    POLICY_VIOLATION = "policy_violation"  # Policy engine triggered
    RESOURCE_EXHAUSTION = "resource_exhaustion"  # System resource abuse
    ANOMALY_DETECTED = "anomaly_detected"  # Behavioral anomaly
    TRUST_THRESHOLD = "trust_threshold"    # Trust score too low
    TIMEOUT = "timeout"                   # Execution timeout
    EXCEPTION = "exception"              # Unhandled exception
    USER_REQUEST = "user_request"        # End user request
    MAINTENANCE = "maintenance"          # Planned maintenance

# Trigger configurations
TRIGGER_CONFIGS = {
    KillSwitchCategory.MANUAL: TriggerConfig(
        requires_approval=False,  # Immediate for manual
        timeout_seconds=0,
        notification=True
    ),
    KillSwitchCategory.POLICY_VIOLATION: TriggerConfig(
        requires_approval=False,  # Immediate for violations
        timeout_seconds=0,
        notification=True
    ),
    KillSwitchCategory.RESOURCE_EXHAUSTION: TriggerConfig(
        requires_approval=False,
        timeout_seconds=0,
        notification=True
    ),
    KillSwitchCategory.TRUST_THRESHOLD: TriggerConfig(
        requires_approval=False,
        timeout_seconds=0,
        notification=True
    ),
    KillSwitchCategory.TIMEOUT: TriggerConfig(
        requires_approval=False,
        timeout_seconds=0,
        notification=True
    ),
}
```

## Process Termination

```python
class ProcessKiller:
    """Terminates agent process trees."""
    
    def terminate_tree(
        self,
        processes: List[ProcessInfo],
        signal: signal.Signals = signal.SIGKILL
    ) -> TerminationResult:
        """Kill a process and all its children."""
        
        terminated = []
        failed = []
        
        # Build process tree
        tree = self._build_process_tree(processes)
        
        # Kill in reverse tree order (children first)
        for process in reversed(tree):
            try:
                os.killpg(os.getpgid(process.pid), signal)
                terminated.append(process)
            except ProcessLookupError:
                # Already dead
                terminated.append(process)
            except Exception as e:
                failed.append(ProcessKillError(process.pid, str(e)))
        
        return TerminationResult(
            terminated=terminated,
            failed=failed
        )
    
    def _build_process_tree(
        self, 
        processes: List[ProcessInfo]
    ) -> List[ProcessInfo]:
        """Build ordered list of processes (children before parents)."""
        
        # Get full tree including children
        all_processes = set()
        queue = list(processes)
        
        while queue:
            proc = queue.pop(0)
            all_processes.add(proc)
            
            # Find children
            children = self._get_child_processes(proc.pid)
            queue.extend(children)
        
        # Sort by depth (deepest first)
        return sorted(
            all_processes, 
            key=lambda p: self._get_process_depth(p.pid),
            reverse=True
        )
```

## Network Isolation

```python
class NetworkIsolator:
    """Isolates agent from network to prevent communication."""
    
    def isolate(self, agent_id: str) -> None:
        """Immediately isolate agent from network."""
        
        # Get agent's network namespace
        namespace = self.registry.get_network_namespace(agent_id)
        
        if namespace:
            # Apply null route (blackhole)
            self._apply_null_route(namespace)
            
            # Block all egress
            self._block_egress(namespace)
        else:
            # Use iptables fallback
            self._apply_iptables_blackhole(agent_id)
    
    def _apply_null_route(self, namespace: str) -> None:
        """Add null route to agent's network namespace."""
        subprocess.run([
            'ip', 'netns', 'exec', namespace,
            'ip', 'route', 'add', 'blackhole', '0.0.0.0/0'
        ], check=True)
```

## Kill Switch API

```python
class KillSwitchAPI:
    """REST API for kill switch operations."""
    
    @app.post("/v1/agents/{agent_id}/terminate")
    async def terminate_agent(
        agent_id: str,
        reason: KillSwitchReason,
        x_auth_token: str = Header(...),
        force: bool = True
    ) -> TerminationResult:
        """Immediately terminate an agent."""
        
        # Validate auth token
        user = await auth.validate_token(x_auth_token)
        
        # Execute termination
        return await kill_switch_manager.terminate(
            agent_id=agent_id,
            reason=reason,
            triggered_by=user.user_id,
            force=force
        )
    
    @app.post("/v1/agents/batch-terminate")
    async def batch_terminate(
        agent_ids: List[str],
        reason: KillSwitchReason,
        x_auth_token: str = Header(...)
    ) -> BatchTerminationResult:
        """Terminate multiple agents at once."""
        
        results = []
        for agent_id in agent_ids:
            result = await kill_switch_manager.terminate(...)
            results.append(result)
        
        return BatchTerminationResult(results=results)
```

## Authorization

```python
class KillSwitchAuthorization:
    """Controls who can trigger the kill switch."""
    
    # Roles that can trigger kill switch
    ALLOWED_ROLES = {
        Role.SUPER_ADMIN,
        Role.SECURITY_ADMIN,
        Role.SRE_ON_CALL
    }
    
    def _authorize_termination(
        self,
        triggered_by: str,
        agent_id: str
    ) -> bool:
        
        user = self.user_store.get(triggered_by)
        
        if not user:
            return False
        
        if user.role not in self.ALLOWED_ROLES:
            return False
        
        # Check if user has permission for specific agent
        if not self._has_agent_permission(user, agent_id):
            return False
        
        return True
```

## Performance Characteristics

| Operation | Latency |
|-----------|---------|
| Kill switch trigger (local) | < 5 ms |
| Kill switch trigger (remote) | < 50 ms |
| Process tree termination | < 10 ms |
| Network isolation | < 5 ms |
| Full termination (local) | < 20 ms |
| Full termination (remote) | < 100 ms |

## Alerting

```python
class KillSwitchAlerts:
    """Sends alerts when kill switch is triggered."""
    
    def __init__(
        self,
        pagerduty_client: PagerDutyClient,
        slack_client: SlackClient,
        email_client: EmailClient
    ):
        self.pagerduty = pagerduty_client
        self.slack = slack_client
        self.email = email_client
    
    async def send_alert(
        self,
        termination: TerminationResult
    ) -> None:
        
        # Always alert SRE
        await self.pagerduty.create_incident(
            title=f"Kill Switch Triggered: {termination.agent_id}",
            severity="critical",
            details={
                "agent_id": termination.agent_id,
                "reason": termination.reason.category,
                "triggered_by": termination.triggered_by
            }
        )
        
        # Alert Slack channel
        await self.slack.send_message(
            channel="#ai-alerts",
            message=f"🚨 Kill Switch triggered for agent {termination.agent_id}"
        )
```
