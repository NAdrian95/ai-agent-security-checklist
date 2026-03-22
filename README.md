# 🛡️ AI Agent Security Checklist

### The comprehensive security checklist for teams deploying autonomous AI agents in production.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

Autonomous AI agents can execute code, call APIs, read sensitive data, make financial transactions, and operate for hours without human oversight. Most teams ship them with zero security review.

This checklist exists to fix that.

> **Who is this for?** Engineering teams, security engineers, and founders deploying AI agents that take real-world actions — especially in finance, healthcare, legal, and infrastructure.

> **How to use it:** Start with the [Agent Security Maturity Model](#-agent-security-maturity-model) to assess where you are. Then work through each section, prioritising 🔴 **Critical** items first.

---

## 📑 Contents

- [Agent Security Maturity Model](#-agent-security-maturity-model)
- [Prompt Injection Defence](#-prompt-injection-defence)
- [Tool & API Safety](#-tool--api-safety)
- [Data Exfiltration Prevention](#-data-exfiltration-prevention)
- [Autonomous Execution Controls](#-autonomous-execution-controls)
- [Multi-Agent Orchestration Security](#-multi-agent-orchestration-security)
- [Memory & State Security](#-memory--state-security)
- [Filesystem & Environment Isolation](#-filesystem--environment-isolation)
- [Cost & Resource Controls](#-cost--resource-controls)
- [Logging, Monitoring & Auditability](#-logging-monitoring--auditability)
- [Human-in-the-Loop Policies](#-human-in-the-loop-policies)
- [Supply Chain & Dependency Security](#-supply-chain--dependency-security)
- [MCP (Model Context Protocol) Security](#-mcp-model-context-protocol-security)
- [Deployment & Infrastructure](#-deployment--infrastructure)
- [Incident Response](#-incident-response)
- [Real-World Incident Reference](#-real-world-incident-reference)
- [Frameworks & Further Reading](#-frameworks--further-reading)
- [Contributing](#contributing)

---

## 📊 Agent Security Maturity Model

Use this to assess your current posture and plan improvements. Most teams shipping agents today are at Level 1 or 2.

| Level | Name | Description | Key Characteristics |
|:-----:|------|-------------|---------------------|
| **1** | **Ad Hoc** | No formal agent security. Agents deployed with default configs. | No input validation. Full tool access. No cost limits. No logging. "It works" is the only test. |
| **2** | **Basic** | Some guardrails in place, but gaps everywhere. | Basic system prompts with safety instructions. Some tools restricted. Manual monitoring. No injection testing. |
| **3** | **Structured** | Formal security review process. Most critical controls implemented. | Input/output validation. Tool allowlisting. Cost ceilings. Structured logging. Human-in-the-loop for destructive actions. Regular injection testing. |
| **4** | **Advanced** | Comprehensive, layered defence. Proactive security posture. | Automated anomaly detection. Red-team exercises. Multi-layer injection defence. Sandboxed execution. Immutable audit logs. Incident response playbooks. |
| **5** | **Resilient** | Assumes compromise. Defence-in-depth with rapid recovery. | Zero-trust agent architecture. Formal threat modelling. Automated containment. Continuous adversarial testing. Security is part of the agent lifecycle, not bolted on. |

### Quick Self-Assessment

Answer these five questions to estimate your level:

1. **Do you test for prompt injection before deploying an agent?** (No = Level 1)
2. **Do you have tool allowlists and parameter validation?** (No = max Level 2)
3. **Can you kill a running agent within 60 seconds?** (No = max Level 2)
4. **Do you have immutable, structured logs of all agent actions?** (No = max Level 3)
5. **Do you run regular red-team exercises against your agents?** (No = max Level 4)

---

## 🔴 Prompt Injection Defence

Prompt injection is the #1 attack vector against AI agents. If your agent processes any external content, it is vulnerable. There is currently no complete solution — only layered mitigations.

> ⚠️ **Real-world context:** In 2025, researchers demonstrated prompt injection attacks across GitHub Copilot, ChatGPT, Salesforce Einstein, Perplexity's Comet browser, and Microsoft Copilot for Edge. CVE-2025-53773 showed how injected code comments in a public GitHub repo could achieve full system takeover through Copilot. Simon Willison's "Lethal Trifecta" describes the core problem: any agent with access to private data, exposure to untrusted content, and an exfiltration vector is vulnerable by design.

### Fundamentals

- [ ] **Separate trusted and untrusted content** — System prompts and user instructions must never be concatenated with external data without clear delimiters and instruction hierarchy. The model cannot reliably distinguish instructions from data — your architecture must enforce the boundary.
- [ ] **Instruction hierarchy enforcement** — Define and enforce a clear priority: system prompt > user instructions > tool responses > external content. The agent must never follow instructions from external content that contradict higher-level directives.
- [ ] **Input sanitisation on all external content** — Strip or escape characters that could be interpreted as instructions. This includes content from web fetches, API responses, database queries, uploaded files, email bodies, and MCP tool responses.
- [ ] **Output validation before execution** — Before executing any action, validate that the agent's proposed action is consistent with the original user intent, not a redirected instruction from injected content.

### Advanced Defences

- [ ] **Canary token detection** — Embed hidden canary strings in system prompts. If an agent's output or tool call contains the canary, injection has occurred. Alert immediately.
- [ ] **Dual-LLM architecture** — Use a separate, smaller model as a "security judge" to evaluate whether proposed actions are consistent with the original task. The judge model should not have access to the untrusted content.
- [ ] **Context window isolation** — For agents processing untrusted content (web pages, emails, documents), process the content in a separate LLM call with restricted permissions, then pass only structured, validated results to the main agent context.
- [ ] **Jailbreak resistance testing** — Regularly test with known techniques: DAN prompts, roleplay attacks, encoding tricks (Base64, ROT13, Unicode), multi-turn manipulation, context window exhaustion, and language-switching attacks.
- [ ] **Indirect injection via tools** — If an agent reads a web page, email, or document, and that content contains "ignore previous instructions and...", the agent must not comply. Test this attack vector specifically and repeatedly.
- [ ] **Multi-language injection testing** — Injection attempts in non-English languages, mixed scripts, or right-to-left text often bypass filters. Test across all languages your agent may encounter.
- [ ] **Fragmented injection detection** — Attackers can split malicious instructions across multiple inputs, tool responses, or conversation turns. Each fragment appears benign; combined, they form an attack. Monitor for instruction-like patterns across the full conversation context.

---

## 🔧 Tool & API Safety

Every tool an agent can call is an attack surface. Every API endpoint is a potential weapon.

> ⚠️ **Real-world context:** The Langflow RCE (CVE-2025-3248, CVSS 9.8) showed how an unauthenticated code validation endpoint in an AI workflow tool became a remote code execution vulnerability, exploited in the wild. AI workflow builders are not "just apps" — they are privileged execution surfaces.

- [ ] **Principle of least privilege** — Each agent should only have access to the minimum set of tools required for its current task. Never grant blanket tool access. Revoke tools that are not actively needed.
- [ ] **Tool allowlisting, not blocklisting** — Explicitly define which tools are permitted per agent, per task. Blocklisting known-bad tools will always miss new attack vectors.
- [ ] **Parameter validation on every tool call** — Validate all parameters before execution. Type checking, range validation, format validation, schema enforcement. Never pass raw LLM output directly to an API.
- [ ] **Destructive action gating** — Any tool call that modifies, deletes, sends, publishes, or transacts must require explicit confirmation or human approval. This is non-negotiable for financial, messaging, and data-deletion operations.
- [ ] **Rate limiting per tool** — Limit how frequently each tool can be called within a session. An agent stuck in a loop calling an API 1,000 times in a minute is a production incident and a billing event.
- [ ] **Timeout enforcement** — Every tool call must have a hard timeout. An agent waiting indefinitely for an API response blocks the entire execution chain and may leak connection state.
- [ ] **Tool response sanitisation** — Treat all tool responses as untrusted input. API responses, database results, and web content can contain injection payloads. Re-validate before the agent processes them.
- [ ] **Tool call auditing** — Log every tool invocation with: timestamp, agent ID, tool name, full parameters, response (or error), and the reasoning chain that led to the call.
- [ ] **Dry-run mode** — Implement a mode where the agent plans and proposes tool calls without executing them. Essential for testing, debugging, and building trust with new agents.
- [ ] **Tool capability documentation** — Maintain a clear, machine-readable spec of what each tool can do, what it cannot do, and what side effects it has. The agent's understanding of tool capabilities should come from controlled documentation, not from the tool's own description (which can be poisoned).

---

## 🔒 Data Exfiltration Prevention

An agent with access to sensitive data and outbound network access is a data breach waiting to happen.

> ⚠️ **Real-world context:** The Slack AI incident (August 2024) demonstrated how indirect prompt injection in private channels could trick a corporate AI into summarising sensitive conversations and sending summaries to external addresses. The agent believed it was performing a helpful task — it was acting as an insider threat.

- [ ] **Network egress controls** — Restrict which domains and IPs the agent can communicate with. Default deny, explicit allow. This is the single most effective exfiltration prevention control.
- [ ] **No sensitive data in URLs** — Agent-constructed URLs must never contain PII, credentials, API keys, or internal identifiers in query parameters. These leak via referrer headers, server logs, and browser history.
- [ ] **Output filtering for sensitive patterns** — Scan agent outputs for patterns matching credentials (API keys, tokens, passwords), PII (national IDs, credit card numbers, email addresses), and internal identifiers. Filter before they leave the system boundary.
- [ ] **Cross-session data isolation** — Data from one user's session must never be accessible to another user's agent. This includes conversation history, tool outputs, cached results, and temporary files.
- [ ] **File upload/download controls** — Agents should not upload files to external services or download from untrusted sources without explicit user approval and content scanning.
- [ ] **Data classification-aware processing** — Agents handling regulated data (financial records, health data, legal documents, PII) should have additional restrictions on how that data can be processed, stored, transmitted, and surfaced in responses.
- [ ] **Side-channel exfiltration monitoring** — Watch for data being leaked through: encoded content in URLs, steganographic payloads in images, timing-based channels, error messages crafted to contain data, or tool calls that embed data in parameters.

---

## ⚡ Autonomous Execution Controls

The longer an agent runs without oversight, the more damage it can cause. Autonomy must be bounded.

- [ ] **Maximum execution time limits** — Hard timeout on total agent execution time. No agent should run indefinitely. For overnight tasks, set explicit hour limits with checkpoint-and-review gates.
- [ ] **Maximum step/iteration limits** — Cap the number of reasoning steps, tool calls, or loop iterations. An agent in an infinite loop is burning compute, accumulating costs, and potentially causing cascading damage.
- [ ] **Cost ceiling enforcement** — Set a hard maximum spend per session. Include: LLM API costs, tool API costs, compute costs, and any third-party service fees. Kill the agent if the ceiling is breached.
- [ ] **Checkpoint and resume** — For long-running tasks, implement checkpointing so that if an agent is killed (timeout, error, cost limit), work is preserved and can be reviewed by a human before resuming.
- [ ] **Graceful degradation on failure** — When an agent encounters an error, it must fail safely: save state, log the error, notify the operator, and stop. Not retry aggressively, not attempt workarounds, not escalate privileges, not ask a different tool for the same thing.
- [ ] **Kill switch** — Every autonomous agent must have an immediate, reliable kill mechanism accessible to operators. This must work even if the agent is in a tight loop or has spawned sub-processes. Non-negotiable.
- [ ] **Progressive autonomy model** — Start agents with minimal permissions and expand based on demonstrated reliability over time. Never deploy a new agent at maximum autonomy on day one.
- [ ] **Output validation before persistence** — Before an agent writes results to a database, file system, or external service, validate the output against expected schemas and sanity checks. A compromised agent writing poisoned data can cause downstream damage long after the session ends.

---

## 🤝 Multi-Agent Orchestration Security

Multi-agent systems introduce new attack surfaces: agent-to-agent communication, trust delegation, and cascading failures.

- [ ] **Agent identity and authentication** — Each agent in a multi-agent system must have a verifiable identity. Agents should not be able to impersonate other agents or escalate their identity claims.
- [ ] **Inter-agent message integrity** — Messages between agents should be integrity-checked. An attacker who can modify messages between agents can redirect the entire system's behaviour.
- [ ] **Trust boundaries and delegation limits** — Not all agents should trust all other agents equally. Define explicit trust levels: which agents can instruct which others, what actions those instructions can trigger, and what parameters are permitted.
- [ ] **Cascading failure prevention** — If one agent fails or is compromised, it must not bring down the entire system. Implement circuit breakers and bulkheads between agents. A failure in a research agent should not propagate to an execution agent.
- [ ] **Shared state access controls** — If agents share a common state store (database, message queue, shared memory), access controls must prevent one agent from corrupting or reading another agent's data beyond what is explicitly shared.
- [ ] **Deadlock and livelock detection** — Multi-agent systems can deadlock (Agent A waiting for Agent B waiting for Agent A) or livelock (agents passing work back and forth forever). Implement detection with automatic resolution or escalation.
- [ ] **Privilege escalation between agents** — An agent must not be able to gain elevated privileges by instructing another, more privileged agent to perform actions on its behalf. Validate the originating agent's permissions, not just the executing agent's.
- [ ] **Agent-to-agent injection** — A compromised or manipulated agent can attempt to inject instructions into other agents through shared context, message passing, or shared state. Treat inter-agent communication with the same suspicion as external input.

---

## 🧠 Memory & State Security

Agents with persistent memory introduce time-shifted attack surfaces that traditional security models don't cover.

> ⚠️ **Real-world context:** Palo Alto Networks identified persistent memory as a critical fourth element beyond Simon Willison's Lethal Trifecta. OpenClaw stores context in SOUL.md and MEMORY.md files, enabling time-shifted prompt injection: malicious payloads injected into memory on one day can detonate when the agent's state aligns on another. Lakera AI research (2026) demonstrated how poisoned data sources could corrupt an agent's long-term memory, creating persistent false beliefs the agent defended as correct — a "sleeper agent" scenario.

- [ ] **Memory content validation** — All data written to persistent memory must be validated and sanitised. Memory is a persistence vector for injection attacks — poisoned memories can influence all future sessions.
- [ ] **Memory provenance tracking** — Track the source of every memory entry. Was it derived from user input, tool output, external content, or another agent? This provenance determines trust level and should influence how the memory is weighted in future reasoning.
- [ ] **Memory isolation between contexts** — Memories from one user, project, or security domain must not leak into another. Shared memory spaces must have explicit access controls.
- [ ] **Memory integrity auditing** — Periodically audit memory stores for: injected instructions, anomalous entries, entries that contradict known facts, and entries that have been modified since creation.
- [ ] **Memory expiry and rotation** — Implement TTLs on memory entries. Stale memories should age out. This limits the window of exposure for any poisoned memory entry.
- [ ] **Scratchpad and working memory isolation** — An agent's chain-of-thought, reasoning traces, and temporary working notes should be isolated from persistent memory and from other agents' view. Reasoning traces can contain sensitive intermediate data.
- [ ] **State file security** — Configuration files, memory files, and state persistence (JSON, Markdown, databases) must have appropriate file permissions, encryption at rest, and integrity checking. Plaintext credential storage in state files has been a repeated vulnerability in agent frameworks.

---

## 📁 Filesystem & Environment Isolation

Agents with filesystem access can read secrets, modify configurations, and persist malicious payloads across sessions.

> ⚠️ **Real-world context:** OpenClaw operates with the full privileges of its host user — shell access, file system read/write, and OAuth credentials. Over 135,000 instances were found exposed on the public internet with default configurations. Credentials were stored in plaintext Markdown and JSON files.

- [ ] **Sandboxed filesystem** — Agents should operate in a sandboxed filesystem with no access to system files, other users' data, or credential storage. Use containers, chroot, or platform-specific sandboxing.
- [ ] **Read-only by default** — Agent filesystem access should be read-only unless write access is explicitly required and scoped to specific directories with size limits.
- [ ] **No access to credential files** — Agents must never read `.env` files, SSH keys, browser profiles, password stores, cloud credential files (`~/.aws/credentials`, `~/.kube/config`), or any other credential storage. Block these paths explicitly.
- [ ] **Temporary workspace cleanup** — Files created by agents during execution must be cleaned up after the session ends. No persistent state without explicit design and security review.
- [ ] **Container isolation** — Where possible, run agents in containers with: restricted capabilities (`--cap-drop=ALL`), no host network access, read-only root filesystem, minimal mounted volumes, and non-root user.
- [ ] **Process isolation** — Agents should not be able to spawn arbitrary processes, access other running processes, or read process memory. Restrict with seccomp profiles or AppArmor policies.
- [ ] **Working directory scoping** — Agents should only operate within an explicitly defined working directory. Any attempt to access paths outside this directory should be blocked and logged as a security event.

---

## 💰 Cost & Resource Controls

AI agents can generate massive bills through API calls, compute usage, and external service consumption. A runaway agent is a financial incident.

- [ ] **Per-session cost budgets** — Set a hard maximum spend per agent session. Include: LLM API costs, tool API costs, compute costs, and third-party service fees. Terminate the agent if the budget is exceeded.
- [ ] **Per-model cost tracking** — Track costs by which LLM model is being called. Agents that silently escalate to more expensive models (GPT-4, Claude Opus) without justification are wasting money and may indicate unexpected behaviour.
- [ ] **Token usage monitoring** — Monitor input and output token counts. A sudden spike often indicates: the agent is stuck in a loop, processing injected content, hallucinating verbose outputs, or having its context window polluted.
- [ ] **Alerting on cost anomalies** — Set alerts for: cost exceeding 2x expected session cost, more than N tool calls in M minutes, token usage indicating repeated retries, and unexpected model tier escalation.
- [ ] **Resource quotas** — CPU, memory, and disk usage limits. An agent that fills a disk or consumes all available memory affects the entire host system.
- [ ] **Billing isolation** — Use separate API keys or billing accounts for agent workloads. A runaway agent should not affect production billing or exhaust shared rate limits.

---

## 📊 Logging, Monitoring & Auditability

If you can't see what an agent did, you can't secure it, debug it, or trust it.

- [ ] **Full conversation logging** — Log the complete conversation: system prompt, user messages, agent reasoning, tool calls, tool responses, and final output. This is your audit trail and your forensic evidence.
- [ ] **Structured logging format** — Use structured logging (JSON) with consistent fields: `timestamp`, `session_id`, `agent_id`, `event_type`, `content`, `metadata`. Not unstructured text dumps.
- [ ] **Reasoning trace preservation** — Log the full chain-of-thought, scratchpad reasoning, and decision traces, not just the final action. This is critical for debugging injection attacks and understanding failure modes.
- [ ] **Immutable audit logs** — Audit logs must be append-only and stored separately from the agent's accessible storage. An agent should never be able to read, modify, or delete its own audit logs.
- [ ] **Real-time monitoring dashboard** — For production agents: active sessions, tool calls/minute, cost accumulation, error rates, anomaly flags, and alerts for suspicious patterns.
- [ ] **Alerting on suspicious patterns** — Trigger alerts for: repeated failed tool calls, unusual tool call sequences, attempts to access restricted resources, sudden behaviour changes, and attempts to read or modify logs.
- [ ] **Retention policies** — Define how long logs are retained, especially for agents handling regulated data. Comply with GDPR, SOC 2, and sector-specific requirements.
- [ ] **Log injection prevention** — Agents or external inputs should not be able to inject misleading content into log files. Sanitise all logged content to prevent log poisoning attacks.

> ⚠️ **Real-world context:** Eye Security disclosed a log poisoning vulnerability in OpenClaw where attackers could write malicious content to log files via WebSocket requests. Since the agent reads its own logs for troubleshooting, injected content could influence future agent decisions.

---

## 👤 Human-in-the-Loop Policies

Not everything should be automated. Define clearly where humans must be involved.

- [ ] **Irreversible actions require approval** — Any action that cannot be easily undone (sending emails, making payments, deleting data, publishing content, executing trades) must require human confirmation.
- [ ] **Confidence-based escalation** — When an agent is uncertain (low confidence, ambiguous instructions, conflicting information, novel situation), it must escalate to a human rather than guess.
- [ ] **Periodic review checkpoints** — For long-running agents, insert mandatory human review points at defined intervals. Not optional — mandatory. Build them into the execution pipeline.
- [ ] **Immediate override mechanism** — Humans must be able to override any agent decision at any point, with the override taking effect immediately and the agent acknowledging the override.
- [ ] **Approval timeout defaults to deny** — If a human doesn't respond to an approval request within a defined period, the agent defaults to NOT proceeding. Never auto-approve on timeout.
- [ ] **Escalation chain with fallback** — Define who gets notified when the primary operator is unavailable. An agent blocked on approval with no reachable human is a failed pipeline.
- [ ] **Post-action review for high-stakes domains** — In finance, healthcare, and legal contexts, implement mandatory post-action review even when pre-action approval was granted. Catch errors before they compound.

---

## 📦 Supply Chain & Dependency Security

Your agent is only as secure as the tools, plugins, models, and skills it depends on.

> ⚠️ **Real-world context:** In February 2026, researchers discovered that 12% of skills on OpenClaw's ClawHub marketplace were compromised in the "ClawHavoc" campaign — 341+ malicious skills delivering the Atomic macOS Stealer. By late February, Koi Security found 820+ malicious skills out of 10,700 total (~8%). Malicious skills posed as legitimate tools but contained base64-encoded payloads, hidden MCP server endpoints tunnelled through bore.pub, and fake CLI installers. Trend Micro identified over 2,200 malicious skills on GitHub alone.

- [ ] **Plugin/skill vetting before installation** — If your agent loads external plugins, skills, or tools, vet every single one before deployment. Check: source code, permissions requested, network access patterns, author reputation, and whether the plugin matches its stated purpose.
- [ ] **Model provenance verification** — Know which model you're running, from which provider, which version, and verify its integrity. Model swaps — even minor version changes — can introduce new vulnerabilities or behavioural changes.
- [ ] **Dependency pinning** — Pin all dependencies (LLM SDK versions, tool libraries, model versions) to specific, audited versions. Do not auto-update in production. Test updates in staging first.
- [ ] **Third-party API security audit** — Audit the security posture of every third-party API your agent calls. Their breach becomes your breach. Their downtime becomes your downtime.
- [ ] **Skill/prompt repository security** — If agents load skills or prompts from a shared repository, that repository must have: access controls, code review, integrity checking, and malware scanning. A poisoned skill is a compromised agent.
- [ ] **No execution of install-time prerequisites from skills** — Skills that require running terminal commands during installation should be treated as highly suspicious. "Setup instructions" in a skill file can be a malware delivery vector disguised as documentation.
- [ ] **Automated skill scanning** — Use tools like `mcp-scan`, `skill-scanner`, or equivalent to automatically scan skills and MCP servers for known malicious patterns before installation.

---

## 🔌 MCP (Model Context Protocol) Security

MCP is rapidly becoming the standard for agent-tool integration. It also introduces a new class of vulnerabilities.

> ⚠️ **Real-world context:** The Coalition for Secure AI (CoSAI) released a comprehensive MCP Security whitepaper in January 2026 identifying 12 core threat categories and nearly 40 distinct threats specific to MCP deployments. Over 8,000 MCP servers were found exposed on the public internet in early 2026. 1Password's security team noted that in agent ecosystems, "the line between reading instructions and executing them collapses."

- [ ] **MCP server authentication** — Every MCP server connection must be authenticated. Never trust an MCP server based on network location alone. Default-open MCP servers are remote code execution surfaces.
- [ ] **MCP tool description validation** — Tool descriptions provided by MCP servers can be poisoned to manipulate agent behaviour. Validate tool descriptions against a known-good manifest. Do not allow MCP servers to self-describe capabilities that weren't pre-approved.
- [ ] **MCP transport security** — All MCP connections must use TLS. Verify server certificates. Do not accept self-signed certificates in production.
- [ ] **MCP response sanitisation** — Treat all MCP server responses as untrusted. They can contain prompt injection payloads in tool outputs, error messages, or metadata.
- [ ] **MCP server inventory and auditing** — Maintain an inventory of all MCP servers your agents connect to. Audit regularly. Remove unused connections. Monitor for configuration drift.
- [ ] **MCP rate limiting and cost controls** — MCP tool calls consume resources on both sides. Implement rate limits to prevent abuse and cost overruns.
- [ ] **Localhost binding is not sufficient security** — CVE-2026-25253 (the "ClawJacked" vulnerability) demonstrated that binding to localhost does not prevent exploitation. Malicious websites can open WebSocket connections to localhost services through the user's browser.

---

## 🏗️ Deployment & Infrastructure

How and where you deploy agents matters as much as how you build them.

- [ ] **Separate agent infrastructure** — Agent workloads should run on isolated infrastructure, not alongside production application servers. A compromised agent should not have a lateral movement path to your production environment.
- [ ] **Network segmentation** — Agents should sit in a network segment with restricted access to internal services. No direct access to production databases, internal APIs, or management planes.
- [ ] **Secrets management** — Never hardcode API keys, credentials, or tokens in agent configurations or state files. Use a secrets manager with rotation, access logging, and scoped access policies.
- [ ] **TLS everywhere** — All agent communications must use TLS. To LLM APIs, tools, databases, MCP servers, and other agents. No exceptions.
- [ ] **Immutable deployments** — Deploy agents as immutable artifacts (containers, versioned packages). No runtime modifications to agent code or configuration.
- [ ] **Rollback capability** — Maintain the ability to instantly roll back to a previous agent version. Agent regressions can be security regressions.
- [ ] **Bind to localhost by default** — Any agent gateway or control interface must bind to 127.0.0.1, not 0.0.0.0. Publicly exposed agent gateways are among the most dangerous misconfigurations in the ecosystem.
- [ ] **Authentication on all interfaces** — Every control interface, WebSocket endpoint, API endpoint, and management UI must require authentication. Disable default-open configurations immediately.
- [ ] **Rate limiting on authentication endpoints** — Implement rate limiting and lockout policies on all authentication mechanisms. CVE-2026-25253 was exploitable partly because OpenClaw had no rate limits on password attempts.

---

## 🚨 Incident Response

When — not if — something goes wrong with an agent, you need a plan.

- [ ] **Agent-specific incident playbook** — Document procedures for: compromised agent, data exfiltration, injection attack detected, runaway costs, malicious skill/plugin discovered, and credential exposure.
- [ ] **Immediate containment procedures** — Document how to: kill a running agent, revoke its API keys, isolate its network access, quarantine its filesystem, and preserve logs. All achievable within minutes.
- [ ] **Credential rotation after incidents** — After any agent compromise, rotate all credentials the agent had access to. Assume they were exfiltrated.
- [ ] **Post-incident analysis template** — After every incident, document: what happened, what the agent did, root cause analysis, blast radius assessment, timeline, and remediation actions taken.
- [ ] **Regular red-team exercises** — Schedule periodic adversarial testing specifically targeting agent deployments. Test: injection attacks, privilege escalation, data exfiltration, tool abuse, and multi-agent manipulation.
- [ ] **Communication plan** — Define who gets notified (internal stakeholders, affected users, regulators if applicable) and within what timeframe for different severity levels.
- [ ] **Lessons learned feedback loop** — Ensure findings from incidents and red-team exercises feed back into the checklist, agent configurations, and deployment procedures.

---

## 📋 Real-World Incident Reference

These incidents inform and validate the checklist items above. Study them.

| Date | Incident | Impact | Relevant Sections |
|------|----------|--------|-------------------|
| Aug 2024 | **Slack AI data exfiltration** — Indirect prompt injection in private channels tricked corporate AI into summarising and exfiltrating sensitive conversations. | Sensitive data exposure via AI summarisation | Prompt Injection, Data Exfiltration |
| 2025 | **GitHub Copilot CVE-2025-53773** — Injected code comments in public repos achieved system takeover through Copilot by enabling YOLO mode and executing arbitrary code. | Full system compromise via code assistant | Prompt Injection, Filesystem Isolation |
| 2025 | **Perplexity Comet credential leak** — Browser-based AI agent was tricked via indirect injection on web pages into leaking user credentials. | Credential theft via browser agent | Prompt Injection, Data Exfiltration |
| 2025 | **Langflow RCE (CVE-2025-3248, CVSS 9.8)** — Unauthenticated code injection in AI workflow platform, exploited in the wild. | Remote code execution on agent infrastructure | Deployment, Tool Safety |
| Jan 2026 | **OpenClaw CVE-2026-25253 (CVSS 8.8)** — One-click RCE chain exploitable against even localhost-bound instances via malicious website and WebSocket hijacking. | Full agent and host compromise | Deployment, MCP Security |
| Feb 2026 | **ClawHavoc campaign** — 820+ malicious skills on ClawHub delivering Atomic macOS Stealer. Skills posed as legitimate tools with hidden payloads. | Malware distribution via agent skill supply chain | Supply Chain, Filesystem Isolation |
| Feb 2026 | **OpenClaw log poisoning** — Attackers injected content into agent logs via WebSocket. Agent read its own logs for troubleshooting, executing injected instructions. | Indirect prompt injection via log files | Logging, Prompt Injection, Memory |
| Feb 2026 | **Cursor CVE-2025-59944** — Case sensitivity bug in file path protection allowed attackers to influence agentic behaviour via malicious config files, escalating to RCE. | Remote code execution via config file injection | Filesystem Isolation, Prompt Injection |
| 2026 | **Lakera memory poisoning research** — Demonstrated persistent false beliefs in agents via poisoned data sources. Agents defended injected beliefs as correct. | Long-term agent behaviour manipulation | Memory & State Security |

---

## 📚 Frameworks & Further Reading

- **OWASP Top 10 for LLM Applications 2025** — [owasp.org/www-project-top-10-for-large-language-model-applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — Prompt injection is #1.
- **OWASP Top 10 for Agentic Applications** — Covers the full spectrum of agent-specific risks.
- **CoSAI MCP Security Whitepaper (Jan 2026)** — 12 threat categories and ~40 threats specific to MCP deployments.
- **Simon Willison's "Lethal Trifecta"** — The foundational framework: private data access + untrusted content exposure + exfiltration vector = vulnerable by design.
- **NIST AI Risk Management Framework (AI RMF)** — Federal framework for AI risk management, increasingly referenced in enterprise compliance.
- **ISO 42001** — International standard for AI management systems, now mandating specific controls for prompt injection prevention.
- **Cisco State of AI Security 2026** — Found only 29% of organisations deploying agentic AI were prepared to secure those deployments.

---

## Using This Checklist

### Quick Security Review (30 minutes)
Start with 🔴 **Prompt Injection** and ⚡ **Autonomous Execution Controls**. These are the highest-impact, most commonly missed areas.

### Full Security Audit (half-day)
Work through every section. For each unchecked item, document: applicability, current state, remediation plan, owner, and deadline.

### Continuous Security
Revisit before every major agent deployment or capability expansion. Add new items as new threats emerge. This checklist is a living document.

---

## Contributing

This checklist is a living document. If you've encountered an agent security issue not covered here, or have a better mitigation, contributions are welcome.

1. Fork this repo
2. Add or modify items with clear, actionable language
3. Include real-world context or references where applicable
4. Submit a PR explaining why the addition matters

Keep items practical and specific. _"Be careful with security"_ is not a checklist item. _"Validate all tool call parameters against a schema before execution"_ is.

See [CONTRIBUTING.md](CONTRIBUTING.md) for full guidelines.

---

## License

[MIT License](LICENSE). Use this freely. Credit appreciated but not required.

---

**Built by [Adrian](https://github.com/NAdrian95)** — building autonomous AI systems for finance.

**If this is useful, give it a ⭐ and share it with your team.**
