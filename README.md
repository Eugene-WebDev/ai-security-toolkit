# AI Security Toolkit — Claude Code Skills for Secure, Compliant AI Agents

Four skills I use when designing, reviewing or quoting AI agents, chatbots, RAG systems and automations. Each is a working checklist with sources — verified against primary sources (OWASP, the MCP specification, EU and Polish law) on 2026-10-03.

| Skill | Covers |
|---|---|
| [`ai-agent-security`](skills/ai-agent-security/SKILL.md) | Prompt injection (direct and indirect), the lethal trifecta / Agents Rule of Two, six injection-resistant design patterns, tool and permission design, MCP server/client security, output handling, RAG and memory poisoning, multi-agent failures, logging, and a pre-launch red-team checklist — mapped to OWASP Top 10 for LLM Apps 2025 and OWASP Top 10 for Agentic Applications 2026. |
| [`decision-model-guards`](skills/decision-model-guards/SKILL.md) | Small "System One" decision models as fast guards and routers around agents: TypeSafe JEV (hosted, typed questions with calibrated confidence) and Fastino GLiNER2.5-Decide (open-weight, runs on CPU). Confidence-gated routing, prompt-injection screens, pre-tool-call guards for destructive actions, local PII detection before LLM calls, and their limits — a guard is a detection layer, not the security boundary. |
| [`secrets-rotation`](skills/secrets-rotation/SKILL.md) | Where secrets live, OIDC instead of static keys, rotation strategies, a zero-downtime rotation runbook, leaked-key incident response, pre-commit and history scanning, special cases (LLM provider keys, n8n encryption key, webhook HMAC, OAuth refresh tokens, bot tokens), and rules for agents. |
| [`eu-ai-compliance`](skills/eu-ai-compliance/SKILL.md) | GDPR/RODO for LLM systems, the EU AI Act timeline after the Digital Omnibus on AI (2026), provider vs deployer roles, Art. 50 transparency, high-risk classification, the pending GDPR Omnibus, and Poland's AI act (KRiBSI, UODO) — plus what to put in an offer for an EU client. Not legal advice. |

**Use:** copy a skill folder into `~/.claude/skills/` (Claude Code) — or read them as plain Markdown checklists.

**Related:** [ai-engineering-toolkit](https://github.com/Eugene-WebDev/ai-engineering-toolkit) — architecture, LangChain/LangGraph, MCP server building.

---

*Eugene Melnychenko — AI Automation & Integrations Engineer · [linkedin.com/in/eugene-webdev](https://www.linkedin.com/in/eugene-webdev)*
