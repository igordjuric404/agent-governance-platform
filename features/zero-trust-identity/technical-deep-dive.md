# Zero-Trust Identity — Technical Deep Dive

## Architecture

Zero-Trust Identity in Ophanix implements a **cryptographic identity system** based on modern cryptographic primitives. Every agent receives a unique identity upon provisioning, with credentials that support rotation, revocation, and cross-organizational verification.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Identity Architecture                          │
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │   Agent      │────►│   Identity   │────►│   Credential │    │
│  │   Request    │     │   Issuance   │     │   Storage    │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│                              │                    │              │
│                              ▼                    ▼              │
│                      ┌──────────────┐     ┌──────────────┐    │
│                      │    DID       │     │   Key Store  │    │
│                      │   Registry   │     │  (HSM/Vault) │    │
│                      └──────────────┘     └──────────────┘    │
│                              │                                   │
│                              ▼                                   │
│                      ┌──────────────┐                           │
│                      │   SPIFFE     │                           │
│                      │   / SVID     │                           │
│                      └──────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

## Core Identity Module (`agentmesh/identity/`)

### Agent Identity Structure

```python
@dataclass
class AgentIdentity:
    did: str                           # Decentralized Identifier
    public_key: Ed25519PublicKey       # Primary signing key
    key_id: str                        # Key identifier
    trust_score: int                   # 0-1000
    trust_tier: TrustTier              # Enum: VERIFIED, TRUSTED, STANDARD, etc.
    capabilities: List[str]            # Allowed actions
    namespaces: List[str]              # Organizational namespaces
    issued_at: datetime
    expires_at: Optional[datetime]
    revoked: bool = False
```

### Credential Types

```python
class AgentCredentials:
    """Primary credential container for agent identity."""
    
    def __init__(self, identity: AgentIdentity):
        self.identity = identity
        self.signing_key = Ed25519PrivateKey.generate()
        self.encryption_key = ML_DSA_65_Key.generate()  # Quantum-safe
        self.attestation: Optional[Attestation] = None
    
    def sign(self, payload: bytes) -> Signature:
        return self.signing_key.sign(payload)
    
    def verify(self, payload: bytes, signature: Signature) -> bool:
        return self.identity.public_key.verify(payload, signature)
```

## Cryptographic Standards

### Ed25519 Signature Scheme

Ed25519 provides fast, secure signatures with:
- **Key size:** 256 bits
- **Signature size:** 512 bits (64 bytes)
- **Security level:** 128-bit security (equivalent to AES-128)

```python
class Ed25519Identity:
    @staticmethod
    def generate(agent_id: str) -> AgentIdentity:
        private_key = Ed25519PrivateKey.generate()
        public_key = private_key.public_key()
        
        # Derive DID from public key
        did = f"did:mesh:agent:{hashlib.sha256(public_key.bytes()).hexdigest()[:16]}"
        
        return AgentIdentity(
            did=did,
            public_key=public_key,
            key_id=f"{did}/keys/1",
            trust_score=500,  # Default: STANDARD tier
            trust_tier=TrustTier.STANDARD,
            capabilities=[],
            namespaces=[],
            issued_at=datetime.utcnow(),
            expires_at=datetime.utcnow() + timedelta(days=365)
        )
```

### Quantum-Safe Cryptography (ML-DSA-65)

The module also includes **Module-Lattice Digital Signature Algorithm (ML-DSA-65)** for long-term security against quantum computing threats:

```python
class ML_DSA_65_Key:
    """Post-quantum key for long-term credential protection."""
    
    @staticmethod
    def generate() -> MLDSAPrivateKey:
        # ML-DSA-65 parameter set
        # - Public key: 1,952 bytes
        # - Signature: 3,309 bytes
        # - Security level: NIST Level 3 (post-quantum)
        pass
```

## SPIFFE/SVID Integration

Ophanix implements the **SPIFFE (Secure Production Identity Framework for Everyone)** standard for workload identity:

```python
class SPIFFEIdentityProvider:
    """SPIFFE/SVID implementation for agent identity."""
    
    def __init__(self, trust_domain: str):
        self.trust_domain = trust_domain
    
    def issue_svid(self, identity: AgentIdentity) -> SVID:
        # Create SPIFFE Verifiable Identity Document
        svidalias = SVID(
            spiffe_id=f"spiffe://{self.trust_domain}/agent/{identity.did}",
            x509_svid=self._create_x509_svid(identity),
            federation_bundles={}
        )
        return svidalias
    
    def verify_svid(self, svid: SVID) -> bool:
        # Verify SVID signature against trust bundle
        pass
```

## DID (Decentralized Identifier) Support

```python
class MeshDID:
    """Decentralized Identifier for agent mesh."""
    
    @staticmethod
    def create(agent_id: str, public_key: PublicKey) -> str:
        # DID method: did:mesh:<agent-type>:<hash>
        agent_type = "agent"  # Could be: orchestrator, worker, gateway
        key_hash = hashlib.sha256(public_key.bytes()).hexdigest()[:16]
        return f"did:mesh:{agent_type}:{key_hash}"
    
    @staticmethod
    def resolve(did: str) -> Optional[AgentIdentity]:
        # Look up DID in registry
        pass
    
    @staticmethod
    def verify(did: str, signature: bytes, payload: bytes) -> bool:
        # Verify DID ownership via challenge-response
        pass
```

## Credential Lifecycle

### Issuance

```python
class IdentityIssuer:
    """Issues and manages agent identities."""
    
    def issue_identity(
        self,
        agent_id: str,
        capabilities: List[str],
        namespaces: List[str],
        trust_tier: TrustTier = TrustTier.STANDARD
    ) -> AgentIdentity:
        # Generate cryptographic keys
        identity = Ed25519Identity.generate(agent_id)
        
        # Set initial trust score based on tier
        trust_score = self._tier_to_score(trust_tier)
        
        # Create DID document
        did_doc = self._create_did_document(identity)
        
        # Store in registry
        self.registry.store(identity)
        
        # Issue SVID
        svid = self.spire_provider.issue_svid(identity)
        
        return identity
```

### Rotation

```python
class CredentialRotation:
    """Handles periodic credential rotation."""
    
    def rotate_keys(self, identity: AgentIdentity) -> AgentIdentity:
        # Generate new key pair
        new_private_key = Ed25519PrivateKey.generate()
        new_public_key = new_private_key.public_key()
        
        # Create new DID with key rotation
        new_did = MeshDID.create(identity.agent_id, new_public_key)
        
        # Revoke old key
        self.key_store.revoke(identity.key_id)
        
        # Update identity with new keys
        identity.public_key = new_public_key
        identity.key_id = f"{new_did}/keys/2"
        identity.version += 1
        
        return identity
```

### Revocation

```python
class IdentityRevocation:
    """Revokes compromised or deprecated identities."""
    
    def revoke(self, did: str, reason: RevocationReason) -> None:
        # Update revocation registry
        self.registry.revoke(did)
        
        # Publish to CRL (Certificate Revocation List)
        self.crl.publish(did)
        
        # Emit revocation event
        self.event_bus.publish(Event(
            type="identity.revoked",
            payload={"did": did, "reason": reason}
        ))
```

## Trust Model

### Trust Tiers

| Tier | Score Range | Issuance Requirements |
|------|-------------|----------------------|
| VERIFIED_PARTNER | 900-1000 | Hardware attestation, long-term relationship |
| TRUSTED | 700-899 | Verified identity, positive history |
| STANDARD | 500-699 | Valid identity, no issues |
| PROBATIONARY | 300-499 | New or recently restored |
| UNTRUSTED | 0-299 | Restricted or blocked |

### Delegation Chain

```python
class DelegationChain:
    """Tracks delegation of authority between agents."""
    
    def __init__(self):
        self.delegations: List[Delegation] = []
    
    def delegate(
        self,
        from_agent: AgentIdentity,
        to_agent: AgentIdentity,
        capabilities: List[str],
        constraints: DelegationConstraints
    ) -> Delegation:
        delegation = Delegation(
            delegator=from_agent.did,
            delegatee=to_agent.did,
            capabilities=capabilities,
            constraints=constraints,
            issued_at=datetime.utcnow(),
            expires_at=datetime.utcnow() + constraints.ttl
        )
        
        # Sign delegation
        delegation.signature = from_agent.sign(delegation.to_bytes())
        
        self.delegations.append(delegation)
        return delegation
```

## Security Considerations

1. **HSM Integration** — Private keys stored in Hardware Security Modules
2. **Key Derivation** — Hierarchical deterministic key derivation
3. **Attestation** — Hardware-based attestation for agent identity
4. **Audit Logging** — All identity operations logged with non-repudiation
5. **Rate Limiting** — Identity operations rate-limited to prevent abuse

## Performance

| Operation | Latency |
|-----------|---------|
| Identity issuance | ~50 ms |
| Credential verification | ~0.1 ms |
| Delegation creation | ~10 ms |
| Key rotation | ~100 ms |
