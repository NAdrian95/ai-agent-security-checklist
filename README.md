# 🛡️ AI Agent Security Checklist

**A comprehensive, opinionated security checklist for teams deploying autonomous AI agents in production.**

Autonomous AI agents are powerful — and dangerous if deployed carelessly. They can execute code, access APIs, read sensitive data, make financial transactions, and operate for hours without human oversight. Most teams ship agents with zero security review.

This checklist exists to fix that.

> **Who is this for?** Engineering teams, security engineers, and founders deploying AI agents that take real-world actions — especially in finance, healthcare, legal, and infrastructure.

---

## Contents

- [Prompt Injection Defence](#-prompt-injection-defence)
- [Tool & API Safety](#-tool--api-safety)
- [Data Exfiltration Prevention](#-data-exfiltration-prevention)
- [Autonomous Execution Controls](#-autonomous-execution-controls)
- [Multi-Agent Orchestration Security](#-multi-agent-orchestration-security)
- [Filesystem & Environment Isolation](#-filesystem--environment-isolation)
- [Cost & Resource Controls](#-cost--resource-controls)
- [Logging, Monitoring & Auditability](#-logging-monitoring--auditability)
- [Human-in-the-Loop Policies](#-human-in-the-loop-policies)
- [Supply Chain & Dependency Security](#-supply-chain--dependency-security)
- [Deployment & Infrastructure](#-deployment--infrastructure)
- [Incident Response](#-incident-response)
- [Contributing](#contributing)
- [License](#license)

---

## 🔴 Prompt Injection Defence

Prompt injection is the #1 attack vector against AI agents. If your agent processes external content (web pages, emails, documents, user input), it is vulnerable.

- [ ] **Separate trusted and untrusted content** — System prompts and user instructions must never be concatenated with external data without clear delimiters and instruction hierarchy.
- [ ] **Input sanitisation on all external content** — Strip or escape characters that could be interpreted as instructions. This includes content from web fetches, API responses, database queries, uploaded files, and email bodies.
- [ ] **Instruction hierarchy enforcement** — Define and enforce a clear priority: system prompt > user instructions > external content. The agent should never follow instructions from external content that contradict system or user-level directives.
- [ ] **Canary token detection** — Embed hidden canary strings in system prompts. If an agent's output contains the canary, injection has occurred.
- [ ] **Output validation against injection** — Before executing any action, validate that the agent's proposed action is consistent with the original user intent, not a redirected instruction from injected content.
- [ ] **Jailbreak resistance testing** — Regularly test with known jailbreak techniques: DAN prompts, roleplay attacks, encoding tricks (Base64, ROT13, Unicode), multi-turn manipulation, and context window exhaustion.
- [ ] **Multi-language injection testing** — Injection attempts in non-English languages or mixed scripts often bypass filters. Test across languages your agent may encounter.
- [ ] **Indirect injection via tools** — If an agent reads a web page, and that web page contains "ignore previous instructions and...", the agent must not comply. Test this specifically.

---

## 🔧 Tool & API Safety

Agents that call tools and APIs can cause real-world damage. Every tool is an attack surface.

- [ ] **Principle of least privilege** — Each agent should only have access to the minimum set of tools required for its task. Never grant blanket tool access.
- [ ] **Tool allowlisting over blocklisting** — Explicitly define which tools are permitted. Do not rely on blocking known-bad tools — you will miss new ones.
- [ ] **Parameter validation on every tool call** — Validate all parameters before execution. Type checking, range validation, format validation. Never pass raw LLM output directly to an API.
- [ ] **Destructive action confirmation** — Any tool call that modifies, deletes, or sends data (DELETE endpoints, file removal, email sending, financial transactions) must require explicit confirmation or human approval.
- [ ] **Rate limiting per tool** — Limit how frequently each tool can be called within a session. An agent stuck in a loop calling an API 1,000 times in a minute is a production incident.
- [ ] **Tool call auditing** — Log every tool invocation with: timestamp, tool name, parameters, response, and the reasoning that led to the call.
- [ ] **Timeout enforcement** — Every tool call must have a timeout. An agent waiting indefinitely for an API response will block the entire execution chain.
- [ ] **Dry-run mode** — Implement a mode where the agent plans and proposes tool calls without executing them. Essential for testing and debugging.
- [ ] **Tool response sanitisation** — Treat all tool responses as untrusted input. API responses can contain injection payloads, especially from third-party services.

---

## 🔒 Data Exfiltration Prevention

An agent with access to sensitive data and outbound network access is a data breach waiting to happen.

- [ ] **Network egress controls** — Restrict which domains and IPs the agent can communicate with. Default deny, explicit allow.
- [ ] **No sensitive data in URLs** — Agent-constructed URLs must never contain PII, credentials, API keys, or internal identifiers in query parameters. These leak via referrer headers, server logs, and browser history.
- [ ] **Output filtering for sensitive patterns** — Scan agent outputs for patterns matching credentials (API keys, tokens, passwords), PII (SSNs, credit card numbers, email addresses), and internal identifiers before they leave the system.
- [ ] **Cross-session data isolation** — Data from one user's session must never be accessible to another user's agent session. This includes conversation history, tool outputs, and cached results.
- [ ] **Clipboard and copy restrictions** — If agents operate in browser environments, prevent them from reading clipboard contents or other browser storage that may contain sensitive data.
- [ ] **File upload/download controls** — Agents should not upload files to external services or download from untrusted sources without explicit user approval.
- [ ] **Data classification awareness** — Agents handling classified or regulated data (financial records, health data, legal documents) should have additional restrictions on how that data can be processed, stored, and transmitted.

---

## ⚡ Autonomous Execution Controls

The longer an agent runs without oversight, the more damage it can cause. Autonomy must be bounded.

- [ ] **Maximum execution time limits** — Hard timeout on total agent execution time. No agent should run indefinitely. For overnight tasks, set explicit hour limits.
- [ ] **Maximum step/iteration limits** — Cap the number of reasoning steps, tool calls, or loop iterations. An agent in an infinite loop is burning compute and potentially causing harm.
- [ ] **Cost ceiling enforcement** — Set maximum spend per session (API calls, compute, external service costs). Kill the agent if the ceiling is breached.
- [ ] **Checkpoint and resume capability** — For long-running tasks, implement checkpointing so that if an agent is killed (timeout, error, cost limit), work is not lost and can be reviewed before resuming.
- [ ] **Graceful degradation on failure** — When an agent encounters an error, it should fail safely: save state, log the error, notify the operator, and stop. Not retry aggressively, not attempt workarounds, not escalate privileges.
- [ ] **Kill switch** — Every autonomous agent must have an immediate, reliable kill mechanism accessible to operators. This is non-negotiable.
- [ ] **Progressive autonomy** — Start agents with minimal permissions and expand based on demonstrated reliability. Never deploy a new agent at maximum autonomy.

---

## 🤝 Multi-Agent Orchestration Security

Multi-agent systems introduce new attack surfaces: agent-to-agent communication, trust relationships, and cascading failures.

- [ ] **Agent identity and authentication** — Each agent in a multi-agent system must have a verifiable identity. Agents should not be able to impersonate other agents.
- [ ] **Message integrity** — Inter-agent messages should be integrity-checked. An attacker who can modify messages between agents can redirect the entire system.
- [ ] **Trust boundaries** — Not all agents should trust all other agents equally. Define explicit trust levels: which agents can instruct which others, and what actions those instructions can trigger.
- [ ] **Cascading failure prevention** — If one agent fails or is compromised, it should not bring down the entire system. Implement circuit breakers and bulkheads between agents.
- [ ] **Shared state security** — If agents share a common state store (database, message queue, shared memory), access controls must prevent one agent from corrupting or reading another's data beyond what's explicitly shared.
- [ ] **Deadlock detection** — Multi-agent systems can deadlock (Agent A waiting for Agent B which is waiting for Agent A). Implement detection and automatic resolution.
- [ ] **Privilege escalation between agents** — An agent should not be able to gain elevated privileges by instructing another agent to perform actions on its behalf that it couldn't perform directly.

---

## 📁 Filesystem & Environment Isolation

Agents with filesystem access can read secrets, modify configurations, and persist across sessions.

- [ ] **Sandboxed filesystem** — Agents should operate in a sandboxed filesystem with no access to system files, configuration files, or other users' data.
- [ ] **Read-only by default** — Agent filesystem access should be read-only unless write access is explicitly required and scoped to specific directories.
- [ ] **No access to credentials files** — Agents must never read `.env` files, SSH keys, browser profiles, password stores, cloud credential files, or any other credential storage.
- [ ] **Temporary workspace cleanup** — Files created by agents during execution should be cleaned up after the session ends. No persistent state without explicit design.
- [ ] **Container isolation** — Where possible, run agents in containers with restricted capabilities, no host network access, and minimal mounted volumes.
- [ ] **Process isolation** — Agents should not be able to spawn arbitrary processes, access other running processes, or read process memory.

---

## 💰 Cost & Resource Controls

AI agents can generate massive bills through API calls, compute usage, and external service consumption.

- [ ] **Per-session cost budgets** — Set a hard maximum spend per agent session. Include LLM API costs, tool API costs, compute costs, and any third-party service fees.
- [ ] **Per-model cost tracking** — Track costs broken down by which LLM model is being called. Agents that escalate to more expensive models (GPT-4, Claude Opus) without need are wasting money.
- [ ] **Token usage monitoring** — Monitor input and output token counts. A sudden spike often indicates the agent is stuck in a loop, processing injected content, or hallucinating verbose outputs.
- [ ] **Alerting on cost anomalies** — Set up alerts for: cost exceeding 2x the expected session cost, more than N tool calls in M minutes, and token usage exceeding context window by repeated retries.
- [ ] **Resource quotas** — CPU, memory, and disk usage limits. An agent that fills a disk or consumes all available memory affects the entire host.
- [ ] **Billing isolation** — Use separate API keys or billing accounts for agent workloads so that a runaway agent doesn't affect your production billing.

---

## 📊 Logging, Monitoring & Auditability

If you can't see what an agent did, you can't secure it, debug it, or trust it.

- [ ] **Full conversation logging** — Log the complete conversation: system prompt, user messages, agent reasoning, tool calls, tool responses, and final output. This is your audit trail.
- [ ] **Structured logging format** — Use structured logging (JSON) with consistent fields: timestamp, session_id, agent_id, event_type, content, metadata. Not unstructured text dumps.
- [ ] **Reasoning trace preservation** — If your agent uses chain-of-thought or scratchpad reasoning, log the full reasoning trace, not just the final action. This is critical for debugging injection attacks and understanding failures.
- [ ] **Immutable audit logs** — Audit logs should be append-only and stored separately from the agent's accessible storage. An agent should never be able to modify its own logs.
- [ ] **Real-time monitoring dashboard** — For production agents, implement a live dashboard showing: active sessions, tool calls/minute, cost accumulation, error rates, and anomaly flags.
- [ ] **Retention policies** — Define how long logs are retained, especially for agents handling sensitive data. Comply with relevant regulations (GDPR, SOC 2, etc.).
- [ ] **Alerting on suspicious patterns** — Trigger alerts for: repeated failed tool calls, unusual tool call sequences, attempts to access restricted resources, and sudden changes in agent behaviour patterns.

---

## 👤 Human-in-the-Loop Policies

Not everything should be automated. Define where humans must be involved.

- [ ] **Irreversible actions require approval** — Any action that cannot be undone (sending emails, making payments, deleting data, publishing content) must require human confirmation before execution.
- [ ] **Confidence-based escalation** — When an agent is uncertain about an action (low confidence, ambiguous instructions, conflicting information), it should escalate to a human rather than guess.
- [ ] **Periodic human review checkpoints** — For long-running agents, insert mandatory human review points at defined intervals. Not optional — mandatory.
- [ ] **Override mechanism** — Humans must be able to override any agent decision at any point, with the override taking effect immediately.
- [ ] **Approval timeout** — If a human doesn't respond to an approval request within a defined period, the agent should default to NOT proceeding, not auto-approving.
- [ ] **Escalation chain** — Define who gets notified when the primary operator is unavailable. An agent waiting for approval with no human available is a blocked pipeline.

---

## 📦 Supply Chain & Dependency Security

Your agent is only as secure as the tools, plugins, and models it depends on.

- [ ] **Plugin/skill vetting** — If your agent loads external plugins, skills, or tools, vet them for security before deployment. Check for: code quality, permissions requested, data access patterns, and author reputation.
- [ ] **Model provenance** — Know which model you're running, from which provider, and which version. Model swaps (even minor version changes) can introduce new vulnerabilities or behavioural changes.
- [ ] **Dependency pinning** — Pin all dependencies (LLM SDK versions, tool libraries, model versions) to specific versions. Do not auto-update in production.
- [ ] **Third-party API security** — Audit the security posture of every third-party API your agent calls. Their breach is your breach.
- [ ] **Skill/prompt repository security** — If agents load skills or prompts from a shared repository, that repository must have access controls, code review, and integrity checking. A poisoned skill is a compromised agent.

---

## 🏗️ Deployment & Infrastructure

How and where you deploy agents matters as much as how you build them.

- [ ] **Separate agent infrastructure from production** — Agent workloads should run on isolated infrastructure, not alongside your production application servers.
- [ ] **Network segmentation** — Agents should sit in a network segment with restricted access to internal services. They should not have direct access to production databases, internal APIs, or management planes.
- [ ] **Secrets management** — Never hardcode API keys, credentials, or tokens in agent configurations. Use a secrets manager with rotation and access logging.
- [ ] **TLS everywhere** — All agent communications (to LLM APIs, tools, databases, other agents) must use TLS. No exceptions.
- [ ] **Immutable deployments** — Deploy agents as immutable artifacts (containers, versioned packages). No runtime modifications to agent code or configuration.
- [ ] **Rollback capability** — Maintain the ability to instantly roll back to a previous agent version if a deployment introduces issues.

---

## 🚨 Incident Response

When (not if) something goes wrong, you need a plan.

- [ ] **Agent-specific incident playbook** — Create a documented playbook for agent security incidents: compromised agent, data exfiltration, injection attack, runaway costs, etc.
- [ ] **Immediate containment procedures** — Document how to: kill a running agent, revoke its API keys, isolate its network access, and preserve logs for investigation. All within minutes, not hours.
- [ ] **Post-incident analysis template** — After every incident, document: what happened, what the agent did, what the root cause was, what the blast radius was, and what changes prevent recurrence.
- [ ] **Regular red-team exercises** — Schedule periodic red-team exercises specifically targeting your agent deployments. Test injection attacks, privilege escalation, data exfiltration, and social engineering via agent interactions.
- [ ] **Communication plan** — Define who needs to be notified (internal stakeholders, affected users, regulators) and within what timeframe for different severity levels.

---

## Using This Checklist

### For a Quick Security Review
Start with the 🔴 **Prompt Injection** and ⚡ **Autonomous Execution Controls** sections. These are the highest-impact, most commonly missed areas.

### For a Full Security Audit
Work through every section. For each unchecked item, document: whether it's applicable, the current state, the remediation plan, and the owner.

### For Continuous Security
Revisit this checklist before every major agent deployment or capability expansion. Agents evolve — and so do the threats.

---

## Contributing

This checklist is a living document. If you've encountered an agent security issue not covered here, or have a better mitigation for an existing item, contributions are welcome.

1. Fork this repo
2. Add or modify checklist items with clear, actionable language
3. Include context or references where applicable
4. Submit a PR with a description of why the addition matters

Please keep items practical and specific. "Be careful with security" is not a checklist item. "Validate all tool call parameters against a schema before execution" is.

---

## License

MIT License. Use this freely. Credit appreciated but not required.

---

**Built by [Adrian](https://github.com/NAdrian95)** — building autonomous AI systems for finance. Follow for more open-source agent tooling.
