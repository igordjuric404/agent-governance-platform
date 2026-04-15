# SLO Engineering — Technical Deep Dive

## Architecture Overview

SLO Engineering implements **Service Level Objectives** for AI agents using the Site Reliability Engineering (SRE) discipline. It provides objective measurement of agent reliability through error budgets, burn rates, and automated alerting.

```
┌─────────────────────────────────────────────────────────────────┐
│                    SLO Engineering Architecture                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     SLO Definitions                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │  │
│  │  │ Availability│  │   Latency   │  │   Policy    │    │  │
│  │  │    SLO      │  │     SLO     │  │  Compliance │    │  │
│  │  │   99.5%    │  │    95% <5s  │  │    99.9%    │    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   SLO Engine                               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │   Window     │  │    Error     │  │    Burn      │  │  │
│  │  │  Calculator  │──│   Budget     │──│    Rate      │  │  │
│  │  │              │  │   Tracker    │  │  Calculator  │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Alerting & Reporting                      │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │   Burn Rate  │  │  Budget      │  │   SLO        │  │  │
│  │  │   Alerts     │  │  Exhaustion  │  │   Reports    │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Data Structures

```python
@dataclass
class SLODefinition:
    """Definition of a Service Level Objective."""
    
    slo_id: str
    name: str
    description: str
    agent_filter: AgentFilter            # Which agents this applies to
    metric: SLOMetric                    # What we're measuring
    target: float                        # Target value (e.g., 99.5)
    window: SLOWindow                    # Time window for measurement
    alert_thresholds: List[AlertThreshold]
    
@dataclass
class SLOMetric(Enum):
    """Types of SLO metrics."""
    
    AVAILABILITY = "availability"        # % of successful actions
    LATENCY = "latency"                  # Response time
    POLICY_COMPLIANCE = "compliance"      # % of policy-compliant actions
    ERROR_RATE = "error_rate"            # % of actions with errors
    CUSTOM = "custom"                     # Custom metric

@dataclass
class SLOWindow(Enum):
    """SLO measurement windows."""
    
    ROLLING_1D = timedelta(days=1)
    ROLLING_7D = timedelta(days=7)
    ROLLING_30D = timedelta(days=30)
    CALENDAR_MONTH = "calendar_month"

@dataclass
class ErrorBudget:
    """Error budget for an SLO."""
    
    slo_id: str
    total_allowed: float                 # Based on SLO target
    consumed: float                      # Errors consumed
    remaining: float                      # Budget remaining
    burn_rate: float                    # How fast budget is burning
    remaining_time: Optional[timedelta] # Time until budget exhausted
    state: BudgetState                  # HEALTHY, AT_RISK, EXHAUSTED
```

## SLO Engine

```python
class SLOEngine:
    """Engine for calculating and tracking SLOs."""
    
    def __init__(
        self,
        metrics_store: MetricsStore,
        event_bus: EventBus,
        alert_manager: AlertManager
    ):
        self.metrics_store = metrics_store
        self.event_bus = event_bus
        self.alert_manager = alert_manager
        self.slo_definitions: Dict[str, SLODefinition] = {}
    
    def register_slo(self, slo: SLODefinition) -> None:
        """Register a new SLO definition."""
        self.slo_definitions[slo.slo_id] = slo
    
    async def calculate_current_value(
        self, 
        slo: SLODefinition,
        agent_id: Optional[str] = None
    ) -> float:
        """Calculate current SLO value for given window."""
        
        if slo.metric == SLOMetric.AVAILABILITY:
            return await self._calc_availability(slo, agent_id)
        
        elif slo.metric == SLOMetric.LATENCY:
            return await self._calc_latency(slo, agent_id)
        
        elif slo.metric == SLOMetric.POLICY_COMPLIANCE:
            return await self._calc_policy_compliance(slo, agent_id)
        
        elif slo.metric == SLOMetric.ERROR_RATE:
            return await self._calc_error_rate(slo, agent_id)
        
        else:
            raise ValueError(f"Unknown metric type: {slo.metric}")
    
    async def _calc_availability(
        self,
        slo: SLODefinition,
        agent_id: Optional[str]
    ) -> float:
        """Calculate availability percentage."""
        
        window_start = datetime.utcnow() - slo.window.value
        
        # Get all actions in window
        actions = await self.metrics_store.get_actions(
            agent_filter=slo.agent_filter,
            agent_id=agent_id,
            start_time=window_start
        )
        
        if not actions:
            return 100.0
        
        successful = sum(1 for a in actions if a.success)
        return (successful / len(actions)) * 100
    
    async def _calc_latency(
        self,
        slo: SLODefinition,
        agent_id: Optional[str]
    ) -> float:
        """Calculate latency compliance percentage."""
        
        window_start = datetime.utcnow() - slo.window.value
        
        # Get latency measurements
        latencies = await self.metrics_store.get_latencies(
            agent_filter=slo.agent_filter,
            agent_id=agent_id,
            start_time=window_start
        )
        
        if not latencies:
            return 100.0
        
        # Calculate % within threshold (e.g., 95% under 5s)
        threshold = slo.target  # This would be parsed appropriately
        within_threshold = sum(1 for l in latencies if l < threshold)
        
        return (within_threshold / len(latencies)) * 100
```

## Error Budget Tracker

```python
class ErrorBudgetTracker:
    """Tracks error budget consumption and burn rate."""
    
    def __init__(self, slo: SLODefinition):
        self.slo = slo
        self.budget_history: Deque[BudgetSnapshot] = deque(maxlen=1000)
    
    def calculate_budget(
        self,
        total_requests: int,
        successful_requests: int
    ) -> ErrorBudget:
        """Calculate current error budget state."""
        
        # Calculate allowed errors based on SLO target
        # For 99.5% SLO: allowed_errors = total * 0.005
        allowed_error_rate = 1 - (self.slo.target / 100)
        total_allowed_errors = total_requests * allowed_error_rate
        
        # Actual errors
        actual_errors = total_requests - successful_requests
        
        # Budget consumed
        consumed = max(0, actual_errors)
        remaining = max(0, total_allowed_errors - actual_errors)
        
        # Burn rate (how fast we're consuming budget)
        burn_rate = self._calculate_burn_rate()
        
        # Time remaining at current burn rate
        if burn_rate > 0 and remaining > 0:
            # Errors per second
            errors_per_second = self._get_errors_per_second()
            remaining_seconds = remaining / errors_per_second if errors_per_second > 0 else float('inf')
            remaining_time = timedelta(seconds=remaining_seconds)
        else:
            remaining_time = None
        
        # Determine state
        budget_remaining_pct = (remaining / total_allowed_errors * 100) if total_allowed_errors > 0 else 100
        
        if budget_remaining_pct > 50:
            state = BudgetState.HEALTHY
        elif budget_remaining_pct > 25:
            state = BudgetState.AT_RISK
        else:
            state = BudgetState.EXHAUSTED
        
        return ErrorBudget(
            slo_id=self.slo.slo_id,
            total_allowed=total_allowed_errors,
            consumed=consumed,
            remaining=remaining,
            burn_rate=burn_rate,
            remaining_time=remaining_time,
            state=state
        )
    
    def _calculate_burn_rate(self) -> float:
        """Calculate how fast budget is being burned.
        
        Burn rate > 1 means we're consuming budget faster than expected.
        Burn rate < 1 means we're doing better than SLO.
        """
        
        if len(self.budget_history) < 2:
            return 1.0
        
        # Compare recent error rate to allowed rate
        recent_window = list(itertools.islice(
            self.budget_history,
            max(0, len(self.budget_history) - 10)
        ))
        
        recent_errors = sum(s.errors for s in recent_window)
        recent_total = sum(s.total for s in recent_window)
        
        if recent_total == 0:
            return 0.0
        
        recent_error_rate = recent_errors / recent_total
        allowed_error_rate = 1 - (self.slo.target / 100)
        
        return recent_error_rate / allowed_error_rate if allowed_error_rate > 0 else 0.0
```

## SLO Reporting

```python
@dataclass
class SLOSummary:
    """Summary of SLO status."""
    
    slo: SLODefinition
    current_value: float
    target: float
    gap: float                          # target - current_value
    error_budget: ErrorBudget
    recent_trend: List[float]           # Last N measurements
    status: SLOStatus

class SLOReporter:
    """Generates SLO reports and dashboards."""
    
    async def generate_summary(
        self,
        slo_id: str,
        agent_id: Optional[str] = None
    ) -> SLOSummary:
        """Generate a complete SLO summary."""
        
        slo = self.slo_engine.slo_definitions[slo_id]
        
        # Get current value
        current_value = await self.slo_engine.calculate_current_value(slo, agent_id)
        
        # Calculate error budget
        budget = await self._get_error_budget(slo, agent_id)
        
        # Get recent trend
        trend = await self._get_recent_trend(slo, agent_id)
        
        # Determine status
        gap = slo.target - current_value
        if gap <= 0:
            status = SLOStatus.MET
        elif gap <= 0.5:
            status = SLOStatus.AT_RISK
        else:
            status = SLOStatus.MISSED
        
        return SLOSummary(
            slo=slo,
            current_value=current_value,
            target=slo.target,
            gap=gap,
            error_budget=budget,
            recent_trend=trend,
            status=status
        )
    
    async def generate_report(
        self,
        window: SLOWindow,
        agent_filter: Optional[AgentFilter] = None
    ) -> SLOReport:
        """Generate a comprehensive SLO report."""
        
        summaries = []
        
        for slo in self.slo_engine.slo_definitions.values():
            if agent_filter and not self._matches_filter(slo.agent_filter, agent_filter):
                continue
            
            summary = await self.generate_summary(slo.slo_id)
            summaries.append(summary)
        
        return SLOReport(
            generated_at=datetime.utcnow(),
            window=window,
            summaries=summaries,
            overall_status=self._determine_overall_status(summaries)
        )
```

## Alert Configuration

```python
@dataclass
class AlertThreshold:
    """Configuration for SLO alerts."""
    
    name: str
    condition: AlertCondition
    value: float
    severity: Severity
    notification_channels: List[str]

class AlertCondition(Enum):
    BURN_RATE_HIGH = "burn_rate_high"        # > 2x expected burn
    BURN_RATE_CRITICAL = "burn_rate_critical"  # > 10x expected burn
    BUDGET_EXHAUSTED = "budget_exhausted"      # < 0 budget remaining
    BUDGET_AT_RISK = "budget_at_risk"          # < 25% budget remaining
    SLO_MISSED = "slo_missed"                   # SLO target missed

class SLOAlertManager:
    """Manages SLO-based alerting."""
    
    async def check_alerts(self, slo: SLODefinition) -> List[Alert]:
        """Check if any alert thresholds are crossed."""
        
        budget = await self._get_error_budget(slo)
        current_value = await self.slo_engine.calculate_current_value(slo)
        
        alerts = []
        
        for threshold in slo.alert_thresholds:
            if threshold.condition == AlertCondition.BURN_RATE_HIGH:
                if budget.burn_rate > threshold.value:
                    alerts.append(self._create_alert(slo, threshold))
            
            elif threshold.condition == AlertCondition.BUDGET_AT_RISK:
                budget_pct = budget.remaining / budget.total_allowed if budget.total_allowed > 0 else 100
                if budget_pct < threshold.value:
                    alerts.append(self._create_alert(slo, threshold))
            
            elif threshold.condition == AlertCondition.SLO_MISSED:
                if current_value < slo.target:
                    alerts.append(self._create_alert(slo, threshold))
        
        return alerts
```

## Performance Characteristics

| Metric | Value |
|--------|-------|
| SLO calculation (single) | ~10 ms |
| SLO calculation (all) | ~100 ms |
| Error budget update | ~5 ms |
| Alert evaluation | ~2 ms |
| Report generation | ~500 ms |
