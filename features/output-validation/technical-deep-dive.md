# Output Validation — Technical Deep Dive

## Architecture Overview

Output Validation implements **multi-stage content validation** for agent outputs, checking for sensitive data patterns, policy compliance, and quality metrics.

```python
class OutputValidator:
    """Validates agent outputs before release."""
    
    def __init__(
        self,
        pattern_matcher: PatternMatcher,
        policy_engine: PolicyEngine,
        quality_scorer: QualityScorer
    ):
        self.pattern_matcher = pattern_matcher
        self.policy_engine = policy_engine
        self.quality_scorer = quality_scorer
    
    async def validate(
        self,
        output: AgentOutput,
        context: ValidationContext
    ) -> ValidationResult:
        
        violations = []
        
        # Check for sensitive data patterns
        sensitive_findings = await self.pattern_matcher.scan(output.content)
        if sensitive_findings:
            violations.append(OutputViolation(
                type=ViolationType.SENSITIVE_DATA,
                findings=sensitive_findings
            ))
        
        # Check policy compliance
        policy_result = await self.policy_engine.evaluate_output(output, context)
        if not policy_result.compliant:
            violations.append(OutputViolation(
                type=ViolationType.POLICY_VIOLATION,
                findings=policy_result.violations
            ))
        
        # Check quality
        quality = await self.quality_scorer.score(output)
        if quality.score < context.min_quality_threshold:
            violations.append(OutputViolation(
                type=ViolationType.QUALITY_BELOW_THRESHOLD,
                findings={"score": quality.score, "threshold": context.min_quality_threshold}
            ))
        
        return ValidationResult(
            valid=len(violations) == 0,
            violations=violations,
            sanitized_output=await self._sanitize(output, violations) if violations else output.content
        )
```

## Content Governance

```python
class ContentGovernance:
    """Governs what content can appear in outputs."""
    
    SENSITIVE_PATTERNS = [
        (r'\b\d{3}-\d{2}-\d{4}\b', 'SSN'),
        (r'\b\d{16}\b', 'CREDIT_CARD'),
        (r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', 'EMAIL'),
    ]
    
    def scan_for_sensitive_data(self, content: str) -> List[SensitiveFinding]:
        findings = []
        for pattern, data_type in self.SENSITIVE_PATTERNS:
            matches = re.finditer(pattern, content)
            for match in matches:
                findings.append(SensitiveFinding(
                    type=data_type,
                    match=match.group(),
                    position=match.start()
                ))
        return findings
```
