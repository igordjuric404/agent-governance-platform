# Shadow AI Discovery — Technical Deep Dive

## Architecture Overview

Shadow AI Discovery implements **multi-scanner architecture** to find AI agents across processes, configs, and repositories.

```python
class ShadowAIDiscovery:
    """Discovers AI agents across the organization."""
    
    def __init__(
        self,
        scanners: List[AgentScanner],
        inventory: AgentInventory,
        risk_calculator: RiskCalculator
    ):
        self.scanners = scanners
        self.inventory = inventory
        self.risk_calculator = risk_calculator
    
    async def discover(self, scope: DiscoveryScope) -> DiscoveryResult:
        
        discovered_agents = []
        
        # Run all scanners
        for scanner in self.scanners:
            agents = await scanner.scan(scope)
            discovered_agents.extend(agents)
        
        # Deduplicate
        unique_agents = self._deduplicate(discovered_agents)
        
        # Calculate risk scores
        for agent in unique_agents:
            agent.risk_score = await self.risk_calculator.calculate(agent)
        
        # Update inventory
        await self.inventory.sync(unique_agents)
        
        return DiscoveryResult(
            agents=unique_agents,
            total_found=len(unique_agents),
            scan_duration=datetime.utcnow() - scope.start_time
        )

class ProcessScanner:
    """Scans for agent processes on systems."""
    
    async def scan(self, scope: DiscoveryScope) -> List[DiscoveredAgent]:
        agents = []
        
        for host in scope.hosts:
            processes = await self._get_agent_processes(host)
            for proc in processes:
                if self._is_agent_process(proc):
                    agents.append(DiscoveredAgent(
                        source=AgentSource.PROCESS,
                        process_id=proc.pid,
                        name=proc.name,
                        host=host,
                        command_line=proc.cmdline
                    ))
        
        return agents

class GitHubScanner:
    """Scans GitHub repositories for agent definitions."""
    
    async def scan(self, scope: DiscoveryScope) -> List[DiscoveredAgent]:
        agents = []
        
        for repo in scope.repositories:
            patterns = [
                'agent_*.py',
                '*_agent.py',
                'agent_config*.yaml',
                '.agent*'
            ]
            
            for pattern in patterns:
                files = await self.github.search_files(repo, pattern)
                for file in files:
                    if await self._contains_agent_definition(file):
                        agents.append(DiscoveredAgent(
                            source=AgentSource.REPOSITORY,
                            repo=repo,
                            file=file.path,
                            definition_type=self._detect_type(file)
                        ))
        
        return agents
```

## Inventory Management

```python
class AgentInventory:
    """Manages inventory of all discovered agents."""
    
    async def sync(self, agents: List[DiscoveredAgent]) -> None:
        for agent in agents:
            existing = await self.store.get_by_identifier(agent.identifier)
            
            if existing:
                await self._update(existing, agent)
            else:
                await self._register(agent)
    
    async def get_unmanaged_agents(self) -> List[DiscoveredAgent]:
        """Get agents not under governance."""
        all_agents = await self.store.get_all()
        return [a for a in all_agents if not a.is_managed]
```
