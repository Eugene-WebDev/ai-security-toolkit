---
name: eu-ai-compliance
description: Practical GDPR/RODO + EU AI Act + Polish AI act guide for builders of AI automations, chatbots, RAG and agents (status verified 2026-10-03, NOT legal advice) — current AI Act timeline after the Digital Omnibus on AI (in force 27 Jul 2026: high-risk moved to 2 Dec 2027 / 2 Aug 2028; Art. 50 transparency from 2 Aug 2026; watermarking + new prohibitions 2 Dec 2026), provider vs deployer roles, risk classification, Art. 50 chatbot/deepfake disclosure, AI literacy, GPAI, fines; GDPR for LLM systems (legal basis, DPIA, Art. 22, processor DPAs with model vendors, transfers/DPF, retention, data-subject rights in logs and vector stores, Art. 32 security, 72h breaches, EDPB Opinion 28/2024); pending GDPR Omnibus (Art. 88c, not law yet); Poland (ustawa o systemach sztucznej inteligencji, KRiBSI, UODO). Use when scoping or quoting an AI project for an EU client, writing a privacy/AI section of an offer, choosing an LLM vendor/region, designing logging/retention, or when a client asks "is this legal / what do we need for RODO / AI Act".
---

# EU AI compliance for builders (GDPR/RODO + AI Act + Poland)

⚠️ Engineering checklist, not legal advice. Laws here moved a lot in 2025–2026 — re-check dates before quoting them to a client, and send anything high-risk or contentious to a lawyer/DPO.

## 0. Ten-question triage for any AI project

1. Does it process **personal data** (names, emails, voice, IDs, free text from people)? → GDPR applies. Almost always yes.
2. Is the output used to **decide about people** (hiring, credit, insurance, education, access to services, employee monitoring)? → possible **high-risk** (AI Act Annex III) and **Art. 22 GDPR**.
3. Does it **talk to people** or generate content they see? → **Art. 50** disclosure.
4. Does it generate **synthetic audio/image/video/text** published to others? → marking/labelling.
5. Any **prohibited** use (§2)? → stop.
6. Who is **provider** vs **deployer**? (You building for a client: usually client = deployer; whoever puts it on the market under their name = provider.)
7. Which **model vendor + region**? Transfers outside the EEA?
8. What's **logged**, where, for how long, and who can see it?
9. Special-category data (health, biometrics, religion, union, sex life, political views, children)? → much stricter.
10. Is a **DPIA** needed? (§5 — for new AI processing of personal data at scale, assume yes until shown otherwise.)

## 1. AI Act timeline (as of 2026-10-03)

| Date | What applies |
|---|---|
| 1 Aug 2024 | AI Act in force |
| 2 Feb 2025 | **Prohibited practices** (Art. 5) + **AI literacy** (Art. 4) |
| 2 Aug 2025 | **GPAI model** obligations (model providers), governance, penalties framework |
| 27 Jul 2026 | **Digital Omnibus on AI** in force (EP 16 Jun, Council 29 Jun 2026) |
| **2 Aug 2026** | **Art. 50 transparency** (chatbot disclosure, deepfake/emotion-recognition notices), Commission GPAI enforcement powers, most remaining provisions |
| **2 Dec 2026** | Art. 50(2) **machine-readable marking** of synthetic content (grace for systems placed before 2 Aug 2026); **new prohibitions**: non-consensual intimate imagery ("nudifier") apps and CSAM generation |
| **2 Dec 2027** | **High-risk, Annex III** (stand-alone: employment, education, credit, essential services, law enforcement, migration, justice, biometrics) — was Aug 2026 |
| 2 Aug 2028 | High-risk, **Annex I** (AI in regulated products: machinery, medical devices…) |
| 2 Aug 2030 | High-risk systems used by public authorities (legacy) |

Omnibus also: softened **AI literacy** to "support/take measures for" staff literacy rather than guarantee a level; new **small mid-cap** relief (<750 staff, ≤€150M turnover or ≤€129M balance sheet); narrower "safety component"; wider legal basis to process special-category data for **bias detection** with safeguards.

**Fines:** prohibited practices up to €35M / 7% global turnover; most other obligations €15M / 3%; misleading info to authorities €7.5M / 1% (lower caps for SMEs).

## 2. Prohibited (Art. 5) — never build

Manipulative/subliminal techniques causing harm; exploiting vulnerabilities (age, disability, social/economic situation); social scoring; predicting crime from profiling alone; untargeted scraping of facial images for face databases; **emotion recognition at work or in education** (except medical/safety); biometric categorisation inferring sensitive traits; real-time remote biometric ID in public for law enforcement (narrow exceptions); from 2 Dec 2026 nudifier/NCII and CSAM generation.

## 3. Roles and risk classes

- **Provider** — develops the system (or has it developed) and places it on the market / puts it into service under its own name. Most obligations.
- **Deployer** — uses an AI system under its authority in a professional capacity. Typical for clients using a chatbot/agent you built on Claude/GPT.
- Building a custom system for one client: the client is usually both provider (own name, own use) and deployer; you're a contractor. Put the role split in the contract.
- Substantially modifying a high-risk system or rebranding it can make you the provider.

**Risk levels:** prohibited · **high-risk** (Annex I/III — risk management, data governance, logging (Art. 12, keep logs ≥6 months for deployers), human oversight, accuracy/robustness, conformity assessment, registration, FRIA for some deployers) · **limited/transparency** (Art. 50) · minimal (no AI-Act duties beyond literacy). Most business automations (support bots, internal RAG, document extraction, content drafting, lead scoring for marketing) are **limited or minimal** — but HR screening, credit/insurance pricing, exam grading, benefits access are **high-risk**.

## 4. Art. 50 transparency — what to build in (from 2 Aug 2026)

- **Chatbots/voice agents (provider duty):** tell people they're talking to AI at the start, clearly, unless obvious. Voice agent: say it in the greeting. Web chat: visible label in the widget.
- **Synthetic content (provider):** machine-readable marking (metadata/watermark, C2PA-style) for generated audio/image/video/text — from 2 Dec 2026 for pre-August systems. Exempt: assistive editing, short snippets, code.
- **Deployers:** label deepfakes; label AI-generated text published to inform the public on matters of public interest unless it had genuine human editorial review with someone responsible; inform people exposed to emotion recognition/biometric categorisation.
- A voluntary EU **Code of Practice on transparency of AI-generated content** gives presumption of compliance to signatories.

## 5. GDPR / RODO for LLM systems

**Legal basis (Art. 6):** usually contract (6(1)(b)) for delivering the service, or legitimate interest (6(1)(f)) with a documented 3-step LIA (purpose → necessity → balancing). Consent only when it's genuinely free and withdrawable. Special categories need an Art. 9 condition too.

**EDPB Opinion 28/2024 (AI models):** models trained on personal data are **not anonymous by default** — case by case (extraction must be negligible); legitimate interest is possible for development/deployment with the 3-step test; unlawful training can taint later deployment.

**DPIA (Art. 35):** required when likely high risk — new technology + profiling/evaluation, large-scale or sensitive data, systematic monitoring, vulnerable people. For customer-facing AI handling personal data, do one (2–5 pages is fine for low-risk systems). UODO publishes a Polish list of processing types requiring a DPIA. EDPB's April 2025 "AI Privacy Risks & Mitigations – LLMs" report has worked examples (customer-service chatbot, learning assistant, travel planner) to copy the structure from.

**Art. 22 automated decisions:** decisions with legal/similarly significant effect based solely on automated processing → right to human intervention, to contest, and an explanation; needs contract/law/explicit consent. Design "AI recommends, human decides" with a real (not rubber-stamp) reviewer.

**Processors and vendors (Art. 28):**
- Sign the model vendor's **DPA** (Anthropic, OpenAI, Google, Microsoft, ElevenLabs all have one); list them as sub-processors to the client.
- Check: **no training on API data** (default for business APIs — confirm in terms), **retention** of inputs/outputs (vendor defaults are days to ~30 days; zero-data-retention available on request/enterprise for some), **region** (EU data residency options exist for some vendors/clouds).
- Your own stack is a processor chain too: n8n Cloud, Supabase, Vercel, tracing (LangSmith/Langfuse cloud), vector DB.

**Transfers outside the EEA (Ch. V):** US vendors certified under the **EU-US Data Privacy Framework** → adequacy applies. DPF survived the General Court (Latombe, 3 Sep 2025); appeal C-703/25 P pending at the CJEU (ruling not expected before late 2026/2027) — keep SCCs as a fallback in contracts. Non-certified vendors → SCCs + transfer impact assessment.

**Minimisation, retention, rights:**
- Send the model only what the task needs; mask identifiers where possible.
- Set retention for prompts/outputs/logs/vector chunks/conversation memory, and enforce it with jobs, not policy text.
- **Right to erasure/access must reach every copy:** DB rows, conversation logs, tracing tool, vector store chunks + embeddings, long-term agent memory, backups (documented expiry). Store a `subject_id` on chunks and log rows so you can find them.
- Accuracy (Art. 5(1)(d)) & hallucinations: don't let a model state facts about people unverified; give users a correction route.
- Transparency (Art. 13/14): privacy notice must mention AI processing, vendors/categories of recipients, transfers, retention, and Art. 22 logic where relevant.

**Security (Art. 32):** encryption at rest + in transit, pseudonymisation, least-privilege access, unique service identities, backups with restore tests, vulnerability scanning, pen tests, prompt-injection controls (`ai-agent-security`), secrets management (`secrets-rotation`). Auditors want evidence (key rotation logs, access logs, scan reports), not policy PDFs.

**Breaches (Art. 33/34):** notify the DPA (UODO in Poland) within **72h** of becoming aware unless unlikely to result in risk; notify individuals if high risk. A leaked API key with access to personal data, or an agent exfiltrating data via prompt injection, can be a breach. Keep a breach register even for non-notified incidents.

## 6. Pending: GDPR "Digital Omnibus" (not law as of 2026-10-03)

The Nov 2025 Commission proposal to amend GDPR is still in negotiation (separate from the AI Omnibus that was adopted). Headline items: new **Art. 88c** — legitimate interest for developing/operating AI systems with enhanced transparency and an **unconditional right to object**; the definition of personal data stays as is. EDPB/EDPS joint opinion accepts the idea but asks for stronger safeguards (scraping, children). **Don't rely on it in designs or offers yet.**

## 7. Poland

- **Ustawa o systemach sztucznej inteligencji** — signed 24 Jul 2026, published 27 Jul 2026; main part in force **11 Aug 2026**; inspections, proceedings, sanctions-mitigation settlements and penal provisions from **28 Oct 2026**.
- **KRiBSI** (Komisja Rozwoju i Bezpieczeństwa Sztucznej Inteligencji) — national AI market-surveillance authority: supervision, complaints, inspections, sanctions, opinions, sandbox/innovation support.
- **UODO** — data protection authority (RODO), unchanged role; breaches go here (72h).
- RODO = GDPR (same regulation); Polish *ustawa o ochronie danych osobowych* adds procedure and some sector rules. Employee monitoring: Kodeks pracy art. 22² / 22³ limits (and emotion recognition at work is prohibited under the AI Act).
- NIS2 implementation (KSC amendments) may apply to clients in essential/important sectors — security and incident-reporting duties on top.

## 8. What goes into an offer / SOW for an AI project (EU client)

- Roles: who is provider/deployer, controller/processor; you as processor (or sub-processor) with a DPA if you touch production personal data.
- Vendors + regions + retention + "no training on our data" confirmation.
- Art. 50 disclosure built into the UI/voice greeting; AI-content labelling where applicable.
- Logging spec (what, retention, access) and erasure path across logs/vector store/memory.
- Human review for consequential decisions; note if any use case is high-risk (and that obligations land 2 Dec 2027).
- DPIA support as a deliverable or explicitly client-owned.
- Disclaimer for legal-domain bots: "not legal advice"; for health/finance similar.
- Prompt-injection and red-team testing in acceptance criteria.

## Sources
- Digital Omnibus on AI — Orrick summary: https://www.orrick.com/en/Insights/2026/07/EU-AI-Act-Update-Digital-Omnibus-Finalizes-8-Compliance-Changes ; Jones Walker: https://www.joneswalker.com/en/insights/blogs/ai-law-blog/yes-august-2-still-matters-the-eu-approved-a-high-risk-ai-delay-but-most-trans.html?id=102nbon
- Art. 50 FAQ (European Commission) — https://digital-strategy.ec.europa.eu/en/faqs/transparency-obligations-under-article-50-ai-act
- EDPB Opinion 28/2024 — https://www.edpb.europa.eu/system/files/2024-12/edpb_opinion_202428_ai-models_en.pdf
- EDPB AI Privacy Risks & Mitigations – LLMs (Apr 2025) — https://www.edpb.europa.eu/system/files/2025-04/ai-privacy-risks-and-mitigations-in-llms.pdf
- GDPR Omnibus proposal status — https://www.taylorwessing.com/en/global-data-hub/2026/the-digital-omnibus-proposal/gdh----the-digital-omnibus-and-gdpr
- DPF / Latombe — https://www.freshfields.com/en/our-thinking/blogs/technology-quotient/eu-us-data-privacy-framework-survives-its-first-judicial-challenge-but-more-are-102l4m1
- Polish AI act — https://www.prawo.pl/biznes/prezydent-podpisal-ustawe-o-systemach-ai,1541891.html ; https://zglegal.pl/act-on-artificial-intelligence-systems-signed-by-the-president/
- GDPR Art. 32 for engineers (freeCodeCamp, 2026-05-28) — local KB `telegram-article/2026-05-30_li_gdpr-article-32.md`
