# Memory Integrity — Technical Deep Dive

## Architecture Overview

Memory Integrity implements **tamper-evident episodic memory** for AI agents with integrity verification, poisoning detection, and access control.

```python
class MemoryIntegrityManager:
    """Manages integrity verification for agent memory."""
    
    def __init__(
        self,
        hash_chain: HashChain,
        poisoning_detector: PoisoningDetector,
        access_controller: MemoryAccessControl
    ):
        self.hash_chain = hash_chain
        self.poisoning_detector = poisoning_detector
        self.access_controller = access_controller
    
    async def store_memory(
        self,
        agent_id: str,
        memory: EpisodicMemory,
        authorized_by: str
    ) -> MemoryStoreResult:
        
        # Verify authorization
        if not await self.access_controller.can_write(agent_id, authorized_by):
            raise UnauthorizedMemoryAccess(agent_id, authorized_by)
        
        # Check for poisoning
        poisoning_result = await self.poisoning_detector.analyze(memory)
        if poisoning_result.is_poisoned:
            await self._handle_poisoning(agent_id, memory, poisoning_result)
            return MemoryStoreResult(accepted=False, reason="poisoning_detected")
        
        # Compute integrity hash
        memory_hash = self.hash_chain.compute(memory)
        
        # Store with integrity metadata
        await self.memory_store.store(MemoryRecord(
            agent_id=agent_id,
            content=memory.content,
            hash=memory_hash,
            previous_hash=await self._get_latest_hash(agent_id),
            timestamp=datetime.utcnow(),
            authorized_by=authorized_by
        ))
        
        return MemoryStoreResult(accepted=True, hash=memory_hash)
    
    async def verify_integrity(
        self,
        agent_id: str
    ) -> IntegrityVerification:
        
        records = await self.memory_store.get_all(agent_id)
        
        # Verify hash chain
        chain_valid = self.hash_chain.verify(records)
        
        # Check for tampering
        for record in records:
            computed_hash = self.hash_chain.compute(record.content)
            if computed_hash != record.hash:
                return IntegrityVerification(
                    valid=False,
                    tampered=True,
                    tampered_at=record.timestamp,
                    tampered_by=record.authorized_by
                )
        
        return IntegrityVerification(valid=True, chain_valid=chain_valid)
```
