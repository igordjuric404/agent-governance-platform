# Prompt Injection Detection — Technical Deep Dive

## Architecture Overview

Prompt Injection Detection implements a **12-vector analysis system** to detect various prompt injection techniques.

```python
class PromptInjectionDetector:
    """Detects prompt injection attacks across 12 vectors."""
    
    VECTORS = [
        VectorType.HIDDEN_INSTRUCTIONS,
        VectorType.CONTEXT_SWITCHING,
        VectorType.ENCODED_PAYLOAD,
        VectorType.ROLE_PLAYING,
        VectorType.DELIMITER_INJECTION,
        VectorType.FUTURE_INSTRUCTION,
        VectorType.TEMPLATE_INJECTION,
        VectorType.CONTEXT_OVERFLOW,
        VectorType.CHARACTER_ESCAPE,
        VectorType.MULTILINGUAL_INJECTION,
        VectorType.REAL_INJECTION,
        VectorType.STYLE_INJECTION,
    ]
    
    async def analyze(
        self,
        input_text: str,
        context: InjectionContext
    ) -> InjectionAnalysis:
        
        findings = []
        
        for vector in self.VECTORS:
            result = await self._analyze_vector(input_text, vector, context)
            if result.detected:
                findings.append(result)
        
        # Calculate overall risk
        risk_score = self._calculate_risk_score(findings)
        
        # Determine if input should be blocked
        should_block = risk_score >= self.threshold or any(
            f.vector == VectorType.REAL_INJECTION for f in findings
        )
        
        return InjectionAnalysis(
            input_text=input_text,
            findings=findings,
            risk_score=risk_score,
            should_block=should_block,
            analyzed_at=datetime.utcnow()
        )
    
    async def _analyze_vector(
        self,
        text: str,
        vector: VectorType,
        context: InjectionContext
    ) -> VectorResult:
        
        if vector == VectorType.HIDDEN_INSTRUCTIONS:
            return await self._detect_hidden_instructions(text)
        elif vector == VectorType.CONTEXT_SWITCHING:
            return await self._detect_context_switching(text)
        elif vector == VectorType.ENCODED_PAYLOAD:
            return await self._detect_encoded_payloads(text)
        # ... similar for other vectors
```

## Detection Patterns

```python
class HiddenInstructionDetector:
    """Detects hidden instructions in text."""
    
    PATTERNS = [
        # Instructions after ignorePrevious
        r'(?i)ignore\s+(all\s+)?(previous|prior|above)\s+(instructions?|commands?|rules?)',
        # Instructions after system prompt override
        r'(?i)(system\s+prompt|from now on|you are now)',
        # Delimiter-based injection
        r'(?:---|\.\.\.|===)[[\s\S]*?(?:---|\.\.\.|===)',
        # Hidden instruction patterns
        r'(?i)(forget\s+everything|disregard\s+all\s+previous)',
    ]
    
    async def _detect_hidden_instructions(self, text: str) -> VectorResult:
        matches = []
        for pattern in self.PATTERNS:
            found = re.finditer(pattern, text)
            matches.extend(found)
        
        return VectorResult(
            detected=len(matches) > 0,
            vector=VectorType.HIDDEN_INSTRUCTIONS,
            matches=[m.group() for m in matches],
            confidence=min(1.0, len(matches) * 0.3)
        )
```
