# Rogue Agent Detection — Technical Deep Dive

## Architecture Overview

Rogue Agent Detection implements **behavioral anomaly detection** for AI agents. It establishes baselines of normal behavior per agent and detects deviations that may indicate compromise, goal drift, or malfunction.

```
┌─────────────────────────────────────────────────────────────────┐
│                Rogue Agent Detection Architecture                    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Behavior Baseline                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │  │
│  │  │   Action    │  │   Resource  │  │    Trust    │   │  │
│  │  │   Patterns  │  │   Access    │  │   Score     │   │  │
│  │  │   Baseline  │  │   Baseline  │  │   Trend     │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Anomaly Detection Engine                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │   Statistical│  │  ML-Based   │  │   Rule-Based │  │  │
│  │  │   Detection │──│  Anomaly    │──│   Detection  │  │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Risk Assessment                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │  │
│  │  │   Risk     │  │   Threat    │  │   Action    │    │  │
│  │  │   Score    │──│   Level    │──│   Trigger   │    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

```python
class RogueAgentDetector:
    """Detects rogue behavior in AI agents."""
    
    def __init__(
        self,
        baseline_store: BaselineStore,
        anomaly_engine: AnomalyEngine,
        risk_calculator: RiskCalculator,
        alert_manager: AlertManager
    ):
        self.baseline_store = baseline_store
        self.anomaly_engine = anomaly_engine
        self.risk_calculator = risk_calculator
        self.alert_manager = alert_manager
    
    async def analyze(
        self,
        agent_id: str,
        action: AgentAction
    ) -> DetectionResult:
        
        # Get behavior baseline for this agent
        baseline = await self.baseline_store.get(agent_id)
        
        # Check for anomalies
        anomalies = await self.anomaly_engine.detect(action, baseline)
        
        if not anomalies:
            return DetectionResult(safe=True)
        
        # Calculate risk
        risk = await self.risk_calculator.calculate(anomalies, baseline)
        
        # Determine action
        if risk.level == RiskLevel.CRITICAL:
            await self._trigger_kill_switch(agent_id, risk)
        elif risk.level == RiskLevel.HIGH:
            await self._trigger_alert(risk)
        
        return DetectionResult(
            safe=False,
            anomalies=anomalies,
            risk=risk
        )
```

## Behavior Baseline

```python
@dataclass
class AgentBaseline:
    """Established behavior baseline for an agent."""
    
    agent_id: str
    established_at: datetime
    window_days: int
    
    # Action patterns
    typical_actions: FrozenSet[str]
    action_frequency: Dict[str, float]
    action_sequence_patterns: List[SequencePattern]
    
    # Resource access
    typical_resources: FrozenSet[str]
    typical_access_times: TimeDistribution
    
    # Trust score
    typical_trust_range: Tuple[int, int]
    trust_trajectory: Trajectory
    
    # Risk indicators
    typical_error_rate: float
    typical_latency_ms: float

class BaselineCalculator:
    """Calculates behavior baselines from historical data."""
    
    async def calculate(
        self,
        agent_id: str,
        window_days: int = 30
    ) -> AgentBaseline:
        
        # Get historical actions
        actions = await self.audit_store.get_actions(
            agent_id=agent_id,
            days=window_days
        )
        
        # Calculate patterns
        typical_actions = self._extract_typical_actions(actions)
        action_frequency = self._calculate_frequencies(actions)
        sequences = self._extract_sequences(actions)
        
        # Calculate resource access patterns
        resources = self._extract_resources(actions)
        access_times = self._calculate_time_distribution(actions)
        
        # Calculate trust patterns
        trust_history = await self._get_trust_history(agent_id, window_days)
        
        return AgentBaseline(
            agent_id=agent_id,
            established_at=datetime.utcnow(),
            window_days=window_days,
            typical_actions=typical_actions,
            action_frequency=action_frequency,
            action_sequence_patterns=sequences,
            typical_resources=resources,
            typical_access_times=access_times,
            typical_trust_range=self._get_trust_range(trust_history),
            trust_trajectory=self._calculate_trajectory(trust_history),
            typical_error_rate=self._calculate_error_rate(actions),
            typical_latency_ms=self._calculate_latency(actions)
        )
```

## Anomaly Detection

```python
class AnomalyEngine:
    """Detects behavioral anomalies."""
    
    async def detect(
        self,
        action: AgentAction,
        baseline: AgentBaseline
    ) -> List[Anomaly]:
        
        anomalies = []
        
        # Check for unusual actions
        if action.type not in baseline.typical_actions:
            anomalies.append(Anomaly(
                type=AnomalyType.UNUSUAL_ACTION,
                severity=self._calculate_action_severity(action, baseline),
                evidence={"action": action.type}
            ))
        
        # Check for unusual resources
        if action.resource not in baseline.typical_resources:
            anomalies.append(Anomaly(
                type=AnomalyType.UNUSUAL_RESOURCE,
                severity=self._calculate_resource_severity(action, baseline),
                evidence={"resource": action.resource}
            ))
        
        # Check for unusual timing
        if not self._is_typical_time(action.timestamp, baseline):
            anomalies.append(Anomaly(
                type=AnomalyType.UNUSUAL_TIME,
                severity=Severity.MEDIUM,
                evidence={"timestamp": action.timestamp.isoformat()}
            ))
        
        # Check for velocity anomalies
        if self._is_velocity_anomaly(action, baseline):
            anomalies.append(Anomaly(
                type=AnomalyType.VELOCITY_ANOMALY,
                severity=Severity.HIGH,
                evidence={"rate": action.frequency}
            ))
        
        # Check for sequence anomalies
        if self._is_sequence_anomaly(action, baseline):
            anomalies.append(Anomaly(
                type=AnomalyType.SEQUENCE_ANOMALY,
                severity=Severity.HIGH,
                evidence={"unexpected_sequence": True}
            ))
        
        return anomalies
```

## Risk Calculation

```python
class RiskCalculator:
    """Calculates risk level from detected anomalies."""
    
    RISK_WEIGHTS = {
        AnomalyType.UNUSUAL_ACTION: 0.3,
        AnomalyType.UNUSUAL_RESOURCE: 0.25,
        AnomalyType.UNUSUAL_TIME: 0.1,
        AnomalyType.VELOCITY_ANOMALY: 0.2,
        AnomalyType.SEQUENCE_ANOMALY: 0.25,
        AnomalyType.TRUST_DROP: 0.4,
        AnomalyType.POLICY_VIOLATION: 0.5,
    }
    
    async def calculate(
        self,
        anomalies: List[Anomaly],
        baseline: AgentBaseline
    ) -> RiskAssessment:
        
        if not anomalies:
            return RiskAssessment(level=RiskLevel.LOW, score=0)
        
        # Calculate weighted risk
        total_risk = sum(
            self.RISK_WEIGHTS.get(a.type, 0.1) * self._severity_to_score(a.severity)
            for a in anomalies
        )
        
        # Normalize to 0-100
        risk_score = min(100, total_risk * 100)
        
        # Determine level
        if risk_score >= 80:
            level = RiskLevel.CRITICAL
        elif risk_score >= 60:
            level = RiskLevel.HIGH
        elif risk_score >= 30:
            level = RiskLevel.MEDIUM
        else:
            level = RiskLevel.LOW
        
        # Get recommended action
        action = self._get_recommended_action(level, anomalies)
        
        return RiskAssessment(
            score=risk_score,
            level=level,
            anomalies=anomalies,
            recommended_action=action
        )
```

## ML-Based Detection

```python
class MLAnomalyDetector:
    """Machine learning-based anomaly detection."""
    
    def __init__(self, model_path: str):
        self.model = self._load_model(model_path)
        self.feature_extractor = FeatureExtractor()
    
    async def detect(
        self,
        action: AgentAction,
        baseline: AgentBaseline
    ) -> float:
        """Returns anomaly score (0 = normal, 1 = anomalous)."""
        
        features = self.feature_extractor.extract(action, baseline)
        
        # Get model prediction
        prediction = self.model.predict_proba([features])[0]
        
        return prediction[1]  # Probability of being anomalous
```

## Alert Actions

```python
class AlertAction(Enum):
    KILL_SWITCH = "kill_switch"
    ESCALATE = "escalate"
    MONITOR = "monitor"
    LOG = "log"

THREAT_LEVEL_ACTIONS = {
    ThreatLevel.CRITICAL: AlertAction.KILL_SWITCH,
    ThreatLevel.HIGH: AlertAction.ESCALATE,
    ThreatLevel.MEDIUM: AlertAction.MONITOR,
    ThreatLevel.LOW: AlertAction.LOG,
}
```
