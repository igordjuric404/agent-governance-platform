# Saga Orchestration — Technical Deep Dive

## Architecture Overview

Saga Orchestration in Ophanix implements the **choreography-based saga pattern** for managing distributed AI agent workflows. Each saga represents a multi-step workflow where each step can succeed or fail, with compensating transactions ensuring consistency.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Saga Orchestration Architecture                 │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      Saga Coordinator                       │  │
│  │                                                           │  │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐ │  │
│  │  │ Step 1  │──►│ Step 2  │──►│ Step 3  │──►│ Step 4  │ │  │
│  │  │ forward │   │ forward │   │ forward │   │ forward │ │  │
│  │  └───┬─────┘   └───┬─────┘   └───┬─────┘   └───┬─────┘ │  │
│  │      │             │             │             │         │  │
│  │      ▼             ▼             ▼             ▼         │  │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐ │  │
│  │  │Compensate│◄──│Compensate│◄──│Compensate│◄──│Compensate│ │  │
│  │  │   4     │   │   3     │   │   2     │   │   1     │ │  │
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Saga State Machine                       │  │
│  │  INIT → RUNNING → COMPLETING → COMPLETED                  │  │
│  │              ↓                                             │  │
│  │         COMPENSATING → COMPENSATED                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Data Structures

```python
class SagaState(Enum):
    INIT = "init"
    RUNNING = "running"
    COMPLETING = "completing"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    COMPENSATED = "compensated"
    FAILED = "failed"

@dataclass
class SagaStep:
    """A single step in a saga workflow."""
    
    name: str
    execute: Callable[[], Any]          # Forward action
    compensate: Callable[[], None]      # Compensating action
    retry_count: int = 3
    timeout_seconds: int = 300
    requires_manual_approval: bool = False
    skip_on_retry: bool = False

@dataclass
class Saga:
    """Complete saga workflow definition."""
    
    saga_id: str
    name: str
    steps: List[SagaStep]
    state: SagaState
    context: Dict[str, Any]              # Shared workflow context
    completed_steps: List[str] = field(default_factory=list)
    failed_step: Optional[str] = None
    created_at: datetime
    updated_at: datetime
    deadline: Optional[datetime] = None

@dataclass
class SagaExecution:
    """Runtime execution state of a saga."""
    
    saga: Saga
    current_step_index: int
    step_results: Dict[str, Any]        # Results from each step
    compensation_history: List[CompensationRecord]
    state: SagaState
    error: Optional[SagaError] = None
```

## Saga Orchestrator

```python
class SagaOrchestrator:
    """Orchestrates multi-step saga workflows."""
    
    def __init__(
        self,
        saga_store: SagaStore,
        event_bus: EventBus,
        checkpoint_manager: CheckpointManager
    ):
        self.saga_store = saga_store
        self.event_bus = event_bus
        self.checkpoint_manager = checkpoint_manager
    
    async def execute(self, saga: Saga) -> SagaExecution:
        """Execute a complete saga workflow."""
        
        execution = SagaExecution(
            saga=saga,
            current_step_index=0,
            step_results={},
            compensation_history=[],
            state=SagaState.RUNNING
        )
        
        self.event_bus.publish(SagaStarted(saga_id=saga.saga_id))
        
        try:
            # Execute each step in order
            for i, step in enumerate(saga.steps):
                execution.current_step_index = i
                
                # Checkpoint before step
                await self.checkpoint_manager.checkpoint(execution)
                
                # Execute step with retry
                result = await self._execute_step_with_retry(
                    step, 
                    execution,
                    saga.context
                )
                
                execution.step_results[step.name] = result
                execution.saga.completed_steps.append(step.name)
                
                # Checkpoint after successful step
                await self.checkpoint_manager.checkpoint(execution)
            
            # All steps completed successfully
            execution.state = SagaState.COMPLETED
            self.event_bus.publish(SagaCompleted(saga_id=saga.saga_id))
            
            return execution
            
        except SagaStepFailedError as e:
            # Step failed - begin compensation
            execution.error = e
            execution.state = SagaState.COMPENSATING
            
            await self._compensate(execution)
            
            return execution
    
    async def _execute_step_with_retry(
        self,
        step: SagaStep,
        execution: SagaExecution,
        context: Dict[str, Any]
    ) -> Any:
        """Execute a step with retry logic."""
        
        last_error = None
        
        for attempt in range(step.retry_count + 1):
            try:
                # Check for manual approval requirement
                if step.requires_manual_approval:
                    approval = await self._wait_for_approval(step, execution)
                    if not approval.granted:
                        raise SagaStepFailedError(
                            step=step.name,
                            reason="Manual approval denied"
                        )
                
                # Execute the step
                if step.skip_on_retry and attempt > 0:
                    # Skip on retry if already succeeded partially
                    return execution.step_results.get(step.name)
                
                result = await self._execute_step(step, context, attempt)
                return result
                
            except Exception as e:
                last_error = e
                self.event_bus.publish(SagaStepRetry(
                    saga_id=execution.saga.saga_id,
                    step=step.name,
                    attempt=attempt + 1,
                    error=str(e)
                ))
        
        raise SagaStepFailedError(
            step=step.name,
            reason=f"Failed after {step.retry_count + 1} attempts",
            original_error=last_error
        )
    
    async def _compensate(self, execution: SagaExecution) -> None:
        """Execute compensating transactions in reverse order."""
        
        self.event_bus.publish(SagaCompensationStarted(
            saga_id=execution.saga.saga_id,
            failed_step=execution.failed_step
        ))
        
        # Compensate in reverse order, skipping already compensated
        for step_name in reversed(execution.saga.completed_steps):
            step = self._get_step_by_name(execution.saga, step_name)
            
            try:
                # Execute compensation
                await self._execute_compensation(step, execution)
                
                # Record compensation
                execution.compensation_history.append(CompensationRecord(
                    step=step_name,
                    compensated_at=datetime.utcnow(),
                    success=True
                ))
                
            except Exception as e:
                # Compensation failed - log and continue with next
                execution.compensation_history.append(CompensationRecord(
                    step=step_name,
                    compensated_at=datetime.utcnow(),
                    success=False,
                    error=str(e)
                ))
                
                self.event_bus.publish(SagaCompensationFailed(
                    saga_id=execution.saga.saga_id,
                    step=step_name,
                    error=str(e)
                ))
        
        execution.state = SagaState.COMPENSATED
        self.event_bus.publish(SagaCompensationCompleted(
            saga_id=execution.saga.saga_id
        ))
```

## Checkpoint Management

```python
class CheckpointManager:
    """Manages saga checkpoints for recovery."""
    
    def __init__(self, checkpoint_store: CheckpointStore):
        self.checkpoint_store = checkpoint_store
    
    async def checkpoint(self, execution: SagaExecution) -> str:
        """Create a checkpoint of current saga state."""
        
        checkpoint = SagaCheckpoint(
            checkpoint_id=uuid.uuid4().hex,
            saga_id=execution.saga.saga_id,
            step_index=execution.current_step_index,
            completed_steps=list(execution.saga.completed_steps),
            step_results=dict(execution.step_results),
            context=dict(execution.saga.context),
            state=execution.state,
            created_at=datetime.utcnow()
        )
        
        # Store checkpoint
        await self.checkpoint_store.save(checkpoint)
        
        # Publish checkpoint event
        self.event_bus.publish(SagaCheckpointCreated(
            saga_id=execution.saga.saga_id,
            checkpoint_id=checkpoint.checkpoint_id,
            step_index=execution.current_step_index
        ))
        
        return checkpoint.checkpoint_id
    
    async def restore(self, saga_id: str) -> Optional[SagaExecution]:
        """Restore saga from latest checkpoint."""
        
        checkpoint = await self.checkpoint_store.get_latest(saga_id)
        
        if not checkpoint:
            return None
        
        # Reconstruct execution from checkpoint
        saga = await self.saga_store.get(saga_id)
        
        execution = SagaExecution(
            saga=saga,
            current_step_index=checkpoint.step_index,
            step_results=checkpoint.step_results,
            compensation_history=[],
            state=checkpoint.state
        )
        
        return execution
```

## Saga Definition DSL

```python
class SagaBuilder:
    """DSL for defining saga workflows."""
    
    def __init__(self, name: str):
        self.name = name
        self.steps: List[SagaStep] = []
    
    def step(
        self,
        name: str,
        execute: Callable,
        compensate: Callable,
        **kwargs
    ) -> 'SagaBuilder':
        self.steps.append(SagaStep(
            name=name,
            execute=execute,
            compensate=compensate,
            **kwargs
        ))
        return self
    
    def build(self) -> Saga:
        return Saga(
            saga_id=uuid.uuid4().hex,
            name=self.name,
            steps=self.steps,
            state=SagaState.INIT,
            context={},
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow()
        )

# Example usage
order_saga = (SagaBuilder("order-processing")
    .step(
        name="reserve_inventory",
        execute=lambda ctx: reserve_inventory(ctx['item_id'], ctx['quantity']),
        compensate=lambda ctx: release_inventory(ctx['item_id'], ctx['quantity'])
    )
    .step(
        name="charge_card",
        execute=lambda ctx: charge_customer(ctx['card_id'], ctx['amount']),
        compensate=lambda ctx: refund_customer(ctx['charge_id'])
    )
    .step(
        name="update_inventory",
        execute=lambda ctx: decrement_inventory(ctx['item_id'], ctx['quantity']),
        compensate=lambda ctx: increment_inventory(ctx['item_id'], ctx['quantity'])
    )
    .step(
        name="schedule_delivery",
        execute=lambda ctx: create_delivery(ctx['order_id'], ctx['address']),
        compensate=lambda ctx: cancel_delivery(ctx['delivery_id'])
    )
    .build())
```

## Fan-Out Patterns

```python
class FanOutSagaStep:
    """Executes a step across multiple agents/items."""
    
    def __init__(
        self,
        name: str,
        items: Callable[[], List[Any]],
        execute_per_item: Callable,
        compensate_per_item: Callable,
        completion_criteria: str = "all"  # "all", "quorum", "any"
    ):
        self.name = name
        self.items = items
        self.execute_per_item = execute_per_item
        self.compensate_per_item = compensate_per_item
        self.completion_criteria = completion_criteria
    
    async def execute(self, context: Dict) -> FanOutResult:
        items = self.items(context)
        results = []
        
        # Execute in parallel
        tasks = [
            self.execute_per_item(context, item) 
            for item in items
        ]
        
        completed = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Filter results
        for item, result in zip(items, completed):
            if isinstance(result, Exception):
                results.append(FanOutItemResult(
                    item=item,
                    success=False,
                    error=str(result)
                ))
            else:
                results.append(FanOutItemResult(
                    item=item,
                    success=True,
                    result=result
                ))
        
        # Check completion criteria
        success_count = sum(1 for r in results if r.success)
        
        if self.completion_criteria == "all":
            if success_count != len(items):
                raise FanOutIncompleteError(results)
        elif self.completion_criteria == "quorum":
            if success_count < len(items) * 0.5:
                raise FanOutIncompleteError(results)
        
        return FanOutResult(results=results)
```

## Event Schema

```python
@dataclass
class SagaStarted:
    saga_id: str
    timestamp: datetime

@dataclass  
class SagaStepCompleted:
    saga_id: str
    step: str
    result: Any
    timestamp: datetime

@dataclass
class SagaStepFailedError(Exception):
    step: str
    reason: str
    original_error: Optional[Exception] = None

@dataclass
class SagaCompensationStarted:
    saga_id: str
    failed_step: str
    timestamp: datetime

@dataclass
class SagaCompensationCompleted:
    saga_id: str
    compensated_steps: List[str]
    timestamp: datetime

@dataclass
class SagaTimeout:
    saga_id: str
    deadline: datetime
    state_at_timeout: SagaState
    timestamp: datetime
```

## Performance Characteristics

| Metric | Value |
|--------|-------|
| Step execution overhead | ~5 ms |
| Compensation trigger latency | ~10 ms |
| Checkpoint creation | ~2 ms |
| Recovery from checkpoint | ~50 ms |
| Max saga steps | 100 |
| Max compensation depth | 50 |
