Below is a governance-focused comparison of Microsoft Agent Governance Toolkit and the seven competitors you listed. Scores are my analyst judgments, from 1 to 5, based on publicly documented capabilities as of April 15, 2026. I evaluated ActiveFence under its current Alice branding, and CalypsoAI using current Calypso/F5 materials, because Alice rebranded in 2026 and CalypsoAI’s current guardrails pages now route through F5 AI Guardrails after F5 announced the acquisition. ([Alice][1])

## Key documented features by vendor

* **Microsoft Agent Governance Toolkit**: deterministic runtime policy enforcement between the agent framework and the actions the agent takes, zero-trust identity, execution sandboxing, MCP security gateway, agent discovery/lifecycle controls, audit/observability, broad framework support, and open-source MIT licensing. ([GitHub][2])
* **ActiveFence / Alice**: WonderBuild pre-launch adversarial testing, WonderFence runtime AI firewall and guardrails, WonderCheck ongoing production red teaming and drift detection, multilingual and multimodal coverage, and compliance alignment. ([Alice][1])
* **Airia**: runtime governance, human-in-the-loop approvals, centralized visibility, audit and observability, AI inventory management, risk classification, compliance reporting, and AI system controls. ([Airia][3])
* **Akamai**: Firewall for AI with prompt injection, sensitive-data and output leakage detection, real-time inspection and policy controls, plus API discovery and behavior analysis. ([Akamai TechDocs][4])
* **Aporia**: real-time guardrails between the LLM and the app, Session Explorer observability, PII detection and masking, and enterprise controls such as SAML SSO, same-cloud deployment, Private Link, and HIPAA/SOC 2 posture. ([Aporia][5])
* **Arthur AI**: agent discovery and governance, continuous evals over traces, built-in guardrails for hallucination, prompt injection, toxicity, sensitive data, and PII, plus real-time dashboards and SaaS/on-prem/AWS/GCP deployment. ([Arthur][6])
* **Calypso AI / F5 AI Guardrails**: runtime security for deployed AI models and agents, adversarial attack protection, DLP and policy enforcement, human-in-the-loop intervention logic, audit-ready observability/logging, and AI red team integration. ([F5, Inc.][7])
* **Enkrypt AI**: continuous red teaming, real-time guardrails, policy engine, MCP Scanner and Secure MCP Gateway, audit evidence packs with runtime receipts, and data risk audit for policy-based data approvals. ([Enkrypt AI][8])

## Comparison table

Scale: **1 = weak/minimal**, **3 = competent**, **5 = strong/market-leading**

| Feature                                 | Microsoft AGT | Alice | Airia | Akamai | Aporia | Arthur AI | Calypso / F5 | Enkrypt AI |
| --------------------------------------- | ------------: | ----: | ----: | -----: | -----: | --------: | -----------: | ---------: |
| Runtime policy enforcement / guardrails |             5 |     5 |     4 |      5 |      5 |         4 |            5 |          5 |
| Agent / tool / MCP governance           |             5 |     3 |     4 |      2 |      3 |         3 |            3 |          5 |
| Identity, least-privilege, approvals    |             5 |     2 |     4 |      2 |      3 |         3 |            3 |          4 |
| Asset discovery / inventory             |             4 |     2 |     5 |      3 |      1 |         4 |            2 |          3 |
| Observability / audit trails            |             4 |     4 |     5 |      3 |      5 |         5 |            4 |          4 |
| Red teaming / continuous evaluation     |             3 |     5 |     2 |      1 |      2 |         4 |            5 |          5 |
| Compliance reporting / GRC support      |             4 |     4 |     5 |      3 |      4 |         3 |            4 |          5 |
| Data protection / PII / DLP             |             2 |     4 |     3 |      4 |      4 |         4 |            4 |          5 |
| Deployment / integration flexibility    |             5 |     4 |     4 |      4 |      4 |         5 |            4 |          4 |
| Open-source / extensibility             |             5 |     1 |     2 |      1 |      2 |         2 |            1 |          2 |

One score that deserves explicit context: **Microsoft AGT gets a low DLP/PII score on purpose**, because Microsoft explicitly says the toolkit governs agent actions, not LLM inputs/outputs or content moderation. ([GitHub][2])

## Gaps and focus areas

### Where Microsoft Agent Governance Toolkit is strongest

AGT is strongest where “agent governance” is literal, action-level governance. Its public materials emphasize deterministic checks before tool calls, resource access, and inter-agent messages execute, combined with zero-trust identity, RBAC/human approvals, sandboxing, MCP/tool filtering, kill switch behavior, and broad framework support. That is a sharper runtime control story than vendors whose center of gravity is mostly content filtering, AI firewalls, or post-hoc monitoring. ([GitHub][2])

AGT also has a clear strategic differentiator in being open-source, MIT-licensed, framework-agnostic, and extensible across multiple languages and frameworks. None of the other vendors in this set currently project that same open, developer-first posture. ([GitHub][2])

### Where AGT is weaker

The biggest public gap is **content-layer safety and enterprise TRiSM breadth**. Alice, Calypso/F5, Enkrypt, Aporia, and Arthur all publicly document stronger stories around prompt/response guardrails, PII/DLP, or continuous evaluation/red teaming. AGT explicitly says it is not a prompt guardrail or content moderation tool, so buyers looking for one-box coverage of unsafe prompts, toxic outputs, PII leakage, and adversarial test automation may see it as incomplete unless it is paired with another layer. ([GitHub][2])

The second gap is **enterprise governance operations**. Airia and Enkrypt are stronger in the public record on AI inventory, compliance reporting, control mapping, approvals, and audit artifacts that legal, risk, and governance teams can consume directly. AGT has strong technical building blocks such as audit trails, compliance modules, and discovery, but it is less visibly packaged as an executive or GRC workflow product. ([Airia][3])

### Who leads what

**Alice, Calypso/F5, and Enkrypt** are the clearest leaders in adversarial testing plus runtime protection. Alice has the cleanest lifecycle story, pre-launch testing, runtime guardrails, and post-launch drift testing. Calypso/F5 pairs runtime controls with AI Red Team and threat intelligence. Enkrypt combines continuous red teaming, runtime guardrails, MCP protection, and compliance evidence generation. ([Alice][1])

**Airia** is strongest on governance operations, centralized visibility, inventory, risk classification, approvals, and audit-ready compliance reporting. **Arthur** and **Aporia** stand out on observability and evaluation. **Akamai** is strongest when the buyer’s anchor problem is securing AI apps and APIs in an existing edge/API security stack, not deep action-level agent governance. ([Airia][3])

**Enkrypt** looks like the closest broad enterprise competitor to watch, because its documented surface area spans red teaming, runtime guardrails, policy-to-control mapping, MCP governance, data policy approvals, and exportable evidence packs. ([Enkrypt AI][8])

## Conclusion

For a startup competition, the clearest positioning is:

1. **Double down on AGT’s real moat**: deterministic action governance, zero-trust agent identity, least-privilege execution, MCP/tool governance, and open-source extensibility. That is where Microsoft is most differentiated. ([GitHub][2])
2. **Close the enterprise packaging gap**: inventory, approval routing, compliance dashboards, and exportable evidence packs are where Airia and Enkrypt make the governance story easier for non-engineering buyers. ([Airia][3])
3. **Add or partner for red teaming and evals**: Alice, Calypso/F5, Arthur, and Enkrypt all show stronger public coverage here. Without that, AGT can look like a powerful kernel that still needs a companion testing layer. ([Alice][1])
4. **Offer an optional DLP/content-safety layer**: even if AGT stays action-first, buyers will still ask for PII leakage control, response filtering, and policy enforcement on prompts/outputs. ([GitHub][2])

Net result: **AGT should not try to become a generic AI firewall**. Its best path is to be the trusted runtime control plane for agents, then add enough governance operations, evidence, and testing hooks to remove the need for a second vendor in enterprise deals.

[1]: https://alice.io/llm "Alice (formerly ActiveFence) | AI Security & LLM Safety Platform"
[2]: https://github.com/microsoft/agent-governance-toolkit/blob/main/README.md "agent-governance-toolkit/README.md at main · microsoft/agent-governance-toolkit · GitHub"
[3]: https://airia.com/ai-platform/governance/ "Governance | Airia"
[4]: https://techdocs.akamai.com/cloud-security/docs/protect-your-ai-apps "Protect your AI apps"
[5]: https://gr-docs.aporia.com/ "Introduction - Aporia"
[6]: https://www.arthur.ai/ "Arthur AI | Ship Reliable AI Agents Fast"
[7]: https://calypsoai.com/components/text-v2/ "F5 AI Guardrails | F5"
[8]: https://www.enkryptai.com/ "Enkrypt AI | World's Most Comprehensive AI Security Platform"
