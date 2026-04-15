# Execution Sandboxing — Technical Deep Dive

## Architecture Overview

Execution Sandboxing in Ophanix implements **multi-layered isolation** for AI agent operations. The architecture combines OS-level isolation primitives with application-level enforcement to create defense-in-depth containment.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Sandboxing Architecture                         │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Agent Process                           │   │
│  │  ┌─────────────────────────────────────────────────────┐│   │
│  │  │              Agent Runtime                            ││   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ ││   │
│  │  │  │   Policy    │  │  Capability │  │   Resource  │ ││   │
│  │  │  │   Engine    │──│   Checker   │──│   Limits    │ ││   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘ ││   │
│  │  └─────────────────────────────────────────────────────┘│   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │                                     │
│         ┌───────────────────┼───────────────────┐                │
│         ▼                   ▼                   ▼                │
│  ┌────────────┐     ┌────────────┐     ┌────────────┐         │
│  │   Linux   │     │   gVisor   │     │    Seccomp │         │
│  │ Namespaces│     │  Sandbox   │     │   Filters  │         │
│  └────────────┘     └────────────┘     └────────────┘         │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               Kernel-Level Enforcement                    │   │
│  │  • cgroups (resource limits)                              │   │
│  │  • AppArmor/SELinux (mandatory access control)          │   │
│  │  • seccomp (syscall filtering)                           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Sandbox Provider Interface

```python
class SandboxProvider(ABC):
    """Abstract interface for sandbox implementations."""
    
    @abstractmethod
    def create_sandbox(
        self,
        config: SandboxConfig
    ) -> SandboxInstance:
        """Create a new sandboxed execution environment."""
        pass
    
    @abstractmethod
    def destroy_sandbox(self, instance: SandboxInstance) -> None:
        """Destroy a sandboxed environment."""
        pass

@dataclass
class SandboxConfig:
    """Configuration for sandbox creation."""
    
    agent_id: str
    isolation_level: IsolationLevel          # NONE, PROCESS, CONTAINER, VM
    resource_limits: ResourceLimits
    allowed_syscalls: List[str]               # For seccomp
    allowed_paths: List[str]                  # Filesystem restrictions
    network_mode: NetworkMode                 # NONE, BRIDGE, HOST
    timeout_seconds: int
```

## Sandbox Implementations

### Process-Level Isolation

```python
class ProcessSandbox(SandboxProvider):
    """Lightweight process-based sandboxing."""
    
    def create_sandbox(self, config: SandboxConfig) -> SandboxInstance:
        # Create restricted process with reduced privileges
        process = subprocess.Popen(
            [config.agent_binary],
            preexec_fn=self._apply_restrictions,
            env=self._create_restricted_env(config),
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        
        return SandboxInstance(
            pid=process.pid,
            sandbox_type=SandboxType.PROCESS,
            created_at=datetime.utcnow(),
            config=config
        )
    
    def _apply_restrictions(self):
        # Drop privileges
        os.setgid(config.gid)
        os.setuid(config.uid)
        
        # Apply resource limits
        resource.setrlimit(resource.RLIMIT_CPU, (60, 60))  # 60s CPU
        resource.setrlimit(resource.RLIMIT_MEM, (1e9, 1e9))  # 1GB memory
        
        # Block dangerous syscalls via seccomp
        self._apply_seccomp_filter()
```

### Container-Based Isolation

```python
class ContainerSandbox(SandboxProvider):
    """Docker/container-based sandboxing."""
    
    def create_sandbox(self, config: SandboxConfig) -> SandboxInstance:
        container_config = {
            'Image': config.base_image,
            'CpuShares': config.resource_limits.cpu_share,
            'Memory': config.resource_limits.memory_bytes,
            'NetworkMode': config.network_mode.value,
            'ReadonlyRootfs': True,
            'CapDrop': ['ALL'],  # Drop all capabilities
            'SecurityOpt': ['no-new-privileges'],
        }
        
        container = self.docker_client.containers.run(
            **container_config,
            detach=True
        )
        
        return SandboxInstance(
            container_id=container.id,
            sandbox_type=SandboxType.CONTAINER,
            created_at=datetime.utcnow(),
            config=config
        )
```

### gVisor Integration

```python
class GVisorSandbox(SandboxProvider):
    """gVisor-based sandbox for stronger isolation."""
    
    def create_sandbox(self, config: SandboxConfig) -> SandboxInstance:
        # Use gVisor's runsc for user-space kernel
        runsc_config = [
            'runsc',
            '--platform=ptrace',  # or 'kvm' for VM-level
            '--fsgooser=false',
            '--strace=false',
            '--rootless=true',
            'run',
            '--bundle', config.bundle_path
        ]
        
        process = subprocess.Popen(
            runsc_config,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        
        return SandboxInstance(
            pid=process.pid,
            sandbox_type=SandboxType.GVISOR,
            created_at=datetime.utcnow(),
            config=config
        )
```

## Resource Limits

```python
@dataclass
class ResourceLimits:
    """Resource constraints for sandboxed execution."""
    
    max_cpu_seconds: float = 60.0
    max_memory_bytes: int = 1 * 1024 * 1024 * 1024  # 1GB
    max_disk_bytes: int = 10 * 1024 * 1024 * 1024    # 10GB
    max_network_bandwidth: int = 100 * 1024 * 1024   # 100Mbps
    max_file_descriptors: int = 1024
    max_processes: int = 10
    max_threads_per_process: int = 100

class ResourceEnforcer:
    """Enforces resource limits within sandbox."""
    
    def __init__(self, instance: SandboxInstance):
        self.instance = instance
        self.limits = instance.config.resource_limits
        self.current_usage = ResourceUsage()
    
    def check_limits(self, operation: str) -> bool:
        """Check if operation would exceed limits."""
        
        if operation == 'memory_alloc':
            if self.current_usage.memory + self.alloc_size > self.limits.max_memory_bytes:
                return False
        
        if operation == 'file_create':
            if self.current_usage.disk + self.file_size > self.limits.max_disk_bytes:
                return False
        
        if operation == 'fork':
            if self.current_usage.processes >= self.limits.max_processes:
                return False
        
        return True
    
    def apply_cgroup_limits(self, pid: int):
        """Apply cgroup resource limits."""
        cgroup_path = f"/sys/fs/cgroup/{self.instance.cgroup_id}"
        
        with open(f"{cgroup_path}/cpu.shares", 'w') as f:
            f.write(str(self.limits.max_cpu_seconds * 1024))
        
        with open(f"{cgroup_path}/memory.limit_in_bytes", 'w') as f:
            f.write(str(self.limits.max_memory_bytes))
```

## System Call Filtering

```python
class SyscallFilter:
    """seccomp-based syscall filtering."""
    
    # Allowed syscalls for typical agent operations
    ALLOWED_SYSCALLS = [
        # File operations
        'read', 'write', 'open', 'close', 'stat', 'fstat',
        'lstat', 'poll', 'lseek', 'readlink', 'mkdir', 'rmdir',
        'getdents', 'truncate', 'getcwd',
        
        # Memory
        'mmap', 'mprotect', 'munmap', 'brk',
        
        # Process
        'exit', 'getpid', 'clone', 'wait4', 'execve',
        
        # Networking
        'socket', 'connect', 'accept', 'sendto', 'recvfrom',
        'sendmsg', 'recvmsg', 'shutdown', 'bind', 'listen',
        
        # Time
        'clock_gettime', 'gettimeofday', 'nanosleep',
        
        # Signal
        'rt_sigaction', 'rt_sigreturn', 'rt_sigprocmask',
    ]
    
    DENIED_SYSCALLS = [
        'init_module',    # Load kernel modules
        'delete_module',  # Unload kernel modules
        'ptrace',         # Debugging - potential exploit vector
        'syslog',         # Kernel log access
        'reboot',         # System reboot
        'setuid',         # Privilege escalation
    ]
    
    def generate_seccomp_filter(self) -> List[seccomp.ScmpSyscall]:
        """Generate seccomp filter rules."""
        allow = [seccomp.Syscall(s) for s in self.ALLOWED_SYSCALLS]
        deny = [seccomp.Syscall(s) for s in self.DENIED_SYSCALLS]
        
        return allow
```

## Kill Switch Integration

```python
class SandboxKillSwitch:
    """Instant termination for misbehaving sandboxes."""
    
    def terminate(self, instance: SandboxInstance, reason: str) -> None:
        """Immediately terminate sandboxed execution."""
        
        # Log termination reason
        self.audit_logger.log_sandbox_termination(
            agent_id=instance.config.agent_id,
            reason=reason,
            resource_usage=self.get_usage_stats(instance)
        )
        
        if instance.sandbox_type == SandboxType.CONTAINER:
            # Kill container
            container = self.docker_client.containers.get(instance.container_id)
            container.kill(signal=signal.SIGKILL)
        
        elif instance.sandbox_type == SandboxType.PROCESS:
            # Kill process tree
            os.killpg(os.getpgid(instance.pid), signal.SIGKILL)
        
        # Cleanup resources
        self._cleanup(instance)
```

## Network Isolation

```python
class NetworkIsolation:
    """Network-level isolation for sandboxes."""
    
    def create_network(self, config: SandboxNetworkConfig) -> str:
        if config.mode == NetworkMode.NONE:
            return None  # No network access
        
        if config.mode == NetworkMode.BRIDGE:
            # Create isolated bridge network
            network = self.docker_client.networks.create(
                name=f"sandbox_{config.agent_id}",
                driver="bridge",
                ipam=IPAMConfig(
                    driver="default",
                    pool_config=[PoolConfig(
                        subnet=config.subnet,
                        gateway=config.gateway
                    )]
                )
            )
            return network.id
        
        if config.mode == NetworkMode.WHITELIST:
            # Allow only specific IPs/ports
            return self._create_whitelist_network(config)
```

## Monitoring and Alerting

```python
class SandboxMonitor:
    """Monitors sandbox health and resource usage."""
    
    def check_health(self, instance: SandboxInstance) -> HealthStatus:
        """Check if sandbox is healthy."""
        
        if instance.sandbox_type == SandboxType.CONTAINER:
            container = self.docker_client.containers.get(instance.container_id)
            stats = container.stats(stream=False)
            
            # Check memory threshold
            memory_usage = stats.memory_stats. usage
            memory_limit = stats.memory_stats.limit
            memory_pct = memory_usage / memory_limit
            
            if memory_pct > 0.95:
                return HealthStatus(
                    healthy=False,
                    reason=f"Memory critical: {memory_pct:.1%}",
                    should_terminate=True
                )
            
            # Check if process is still running
            if container.status != 'running':
                return HealthStatus(
                    healthy=False,
                    reason=f"Container not running: {container.status}",
                    should_terminate=False
                )
        
        return HealthStatus(healthy=True)
```

## Performance Characteristics

| Sandbox Type | Startup Latency | Memory Overhead | Isolation Level |
|--------------|-----------------|-----------------|-----------------|
| Process | ~5ms | ~5MB | Low |
| gVisor | ~50ms | ~20MB | High |
| Container | ~200ms | ~50MB | Very High |
| VM | ~2s | ~512MB | Maximum |

## Security Considerations

1. **Defense in Depth** — Multiple isolation layers
2. **Capability Dropping** — All capabilities removed by default
3. **No New Privileges** — Prevent privilege escalation via setuid
4. **Resource Exhaustion Protection** — Prevent DoS attacks
5. **Immutable Root Filesystem** — Prevent filesystem tampering
