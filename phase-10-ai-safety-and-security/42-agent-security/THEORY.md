# Topic 42 — Agent Security

**Why this topic:** Topic 41's attacks matter most when the model can act. An agent is a program whose
control flow is decided by a manipulable component with access to your systems. This topic is how to
grant it real capability without granting real danger.

---

# Part 1 — Theory

## 42.1 The threat model

An agent combines three things that are individually fine and jointly hazardous:

```
untrusted input  (user text, web pages, documents, tool results)
+ private data   (your database, files, credentials)
+ the ability to act  (write, send, execute, spend)
```

Any two are manageable. All three means: **anyone who can get text in front of your agent can
potentially cause actions on your systems.** So the design question is never "will it be
manipulated?" but "what is the worst thing it can do once it is?"

Note the difference from ordinary appsec: your privilege boundary is enforced against a *component*
you deliberately gave broad access to, not just against external requests.

## 42.2 Permissions

Least privilege, applied concretely:

- **Act as the user, not as the system.** The agent's database connection should carry the user's
  permissions. Then a manipulated agent can only reach what that user could reach anyway — which
  converts a breach into a non-event.
- **Scope per session** — this directory, this tenant, this project, this time window.
- **Separate read from write**, and reversible writes from irreversible ones (Topic 28).
- **No ambient credentials.** The agent should never see a token it can print, and never hold
  admin-level access "for convenience".
- **Deny by default.** Capabilities are granted explicitly per session, not inherited from the
  process.

The test of a permission design: assume the agent is fully compromised. List everything it can now
do. If that list is acceptable, your design holds; if it isn't, no prompt will save you.

## 42.3 Sandboxing

Topic 28's controls, now as a security boundary rather than a safety net:

process/container/microVM isolation; no network by default with an explicit allowlist (this alone
prevents most exfiltration); a read-only filesystem plus a scratch area; resource limits; ephemeral
per-session instances; and no credentials in the environment.

Two additions specific to security:

- **Egress control is the highest-value single control.** Data cannot be stolen through a network
  connection that doesn't exist. Allowlist destinations, and block link-local metadata addresses
  (`169.254.169.254`) explicitly — cloud credential theft via SSRF is a standard attack.
- **Assume sandbox escape is possible.** Keep nothing valuable inside the sandbox, so escaping it
  gains little.

## 42.4 Secrets

Rules, in order of importance:

1. **Never in the context window.** Not in the system prompt, not in a tool result, not in an error
   message. If it's in the context, treat it as disclosed.
2. **Injected at the edge.** The tool implementation attaches credentials; the model passes
   parameters, never keys.
3. **Scoped and short-lived** — per tenant where possible, with expiry.
4. **Unprintable.** A tool that dumps environment variables or reads arbitrary config files will
   eventually be asked to. Exclude secret paths by policy.
5. **Rotatable**, because you will one day discover a leak in your logs.

A specific trap: **error messages**. A raw exception often contains a connection string. Sanitize
errors before they enter the context (Topic 26 wanted actionable errors — actionable does not mean
verbose).

## 42.5 Tool authorization

Authorization must be enforced in the tool implementation, on **every** call:

- **Identity from the session, never from arguments.** A `user_id` parameter the model fills in is a
  privilege-escalation vulnerability (Topic 28, Q5). This is the single most common agent security
  bug.
- **Row-level checks** — may *this* user access *this* record? Enforce in the query or the database
  (RLS), not in application logic the agent could influence.
- **Rate and quantity limits** per tool per session — "refund up to $50, at most 3 times".
- **Semantic limits** — an agent authorized to refund should not be able to refund more than the
  order total, regardless of what it computed.
- **Audit every call** with identity, arguments, decision and result.

The framing: **the tool is a public API endpoint and the model is an anonymous internet client.**
Write the authorization you'd write for that.

## 42.6 Human approval

Your last line of defence for irreversible actions, and it must be built to actually work:

- **Show the real action** — the actual arguments, resolved values, and consequences in plain
  language. "Approve tool call?" is not approval; "Send this email to 400 recipients?" is.
- **Approve specifics, not patterns.** "Always allow sending email" recreates the original risk.
- **Make approval unforgeable by the model** — the model must not be able to emit text that your
  system interprets as approval. This sounds obvious and is a real bug class in naive
  implementations.
- **Reject with reason**, fed back as information (Topic 33).
- **Fight fatigue** — approve where consequences are real, auto-allow where they aren't. A user who
  clicks through 30 prompts is providing no security at all.

## 42.7 Monitoring and incident response

Detect: unusual tool sequences, bursts of failed authorizations, injection-classifier hits, egress
attempts to non-allowlisted destinations, cost anomalies, and repeated approval requests for the same
risky action.

Prepare: a **kill switch** per agent/feature, session termination, credential rotation, and an audit
trail good enough to answer "what did it touch?" after the fact. That last question is what an
incident actually consists of — and if your logs can't answer it, the incident has no bottom.

## 42.8 The deployment ladder

How to ship agents responsibly:

1. Read-only, internal users, full logging.
2. Write with approval for everything.
3. Auto-approve a narrow, measured set of safe actions.
4. Widen scope as evidence accumulates.

Never start at step 4 because a demo worked. Capability should follow measured reliability (Phase 9),
not enthusiasm.

---

# Part 2 — Questions to implement

**Attack only your own systems.** Build `agent_security/` here, hardening your Phase 7/8 agent.

### Q1. Worst-case inventory
**Build:** a written list of everything your agent can currently do if fully compromised — every
tool, every credential, every reachable system.
**Explain:** is that list acceptable? Which item frightens you most?

### Q2. Act as the user
**Build:** replace the agent's shared/admin database access with a connection carrying the requesting
user's permissions.
**Check:** an agent asked to read another user's data fails at the database, not in a prompt.
**Explain:** how does this change the impact of a successful injection?

### Q3. The identity-from-arguments bug
**Build:** a tool taking `user_id` as a model argument; craft an injection that changes it.
**Check:** demonstrate cross-user access.
**Explain:** fix it with session-derived identity. Why was the original a vulnerability and not a
prompt weakness?

### Q4. Row-level enforcement
**Build:** enforce access in the database (RLS or a mandatory predicate) rather than in application
code.
**Check:** write a test that bypasses your application layer and still can't read other rows.
**Explain:** why is defence at this layer qualitatively different?

### Q5. Egress control
**Build:** a sandbox with an outbound allowlist. Then have the agent attempt: an allowlisted host, a
random external host, `localhost`, and the cloud metadata address.
**Check:** report which were blocked.
**Explain:** why is egress control the highest-value single control?

### Q6. Secret extraction attempts
**Build:** provide a credential correctly (edge-injected), then attempt extraction: print environment,
read config files, trigger an error containing the connection string, ask directly.
**Check:** report which attempt got closest.
**Explain:** which path did you have to close that you hadn't thought of?

### Q7. Error sanitization
**Build:** an error sanitizer that strips secrets, internal paths and stack traces before errors enter
the context, while remaining actionable for the model.
**Check:** an error that previously leaked a connection string no longer does; the agent can still
self-correct.
**Explain:** how did you keep it actionable?

### Q8. Quantity and semantic limits
**Build:** a refund tool with per-session count and amount caps, plus a semantic rule (never exceed
the order total).
**Check:** attempt to breach each limit via the agent.
**Explain:** which limit would have saved you the most money in a real incident?

### Q9. Unforgeable approval
**Build:** an approval mechanism the model cannot influence. Then try to make the agent produce output
that your system might interpret as approval.
**Check:** it cannot.
**Explain:** describe how a naive implementation could be spoofed.

### Q10. Approval quality
**Build:** two approval prompts for the same action — a raw JSON tool call, and a plain-language
summary with resolved consequences.
**Check:** show both to someone (or judge honestly yourself) and see whether the risk is apparent.
**Explain:** which would you have approved without thinking? What does that tell you?

### Q11. Anomaly detection
**Build:** detection for unusual tool sequences, failed-authorization bursts, blocked egress attempts,
and cost spikes.
**Check:** run your Topic 41 attack suite and confirm alerts fire.
**Explain:** which attack was quietest, and how would you catch it?

### Q12. Kill switch and forensics
**Build:** a per-agent kill switch, session termination, and an audit query answering "what did
session X touch?"
**Check:** run a session doing 20 varied actions, then produce the complete list from the audit trail
alone.
**Explain:** was anything missing from the trail? What would that gap cost during an incident?

### Q13. Red team your own agent
**Build:** 20 attack attempts combining injection, tool abuse, privilege escalation, exfiltration and
cost attacks.
**Check:** report outcomes and which control stopped each.
**Explain:** report your success rate as an attacker. Which single control blocked the most attacks?

### Q14. The deployment decision
**Build:** decide which rung of §42.8's ladder your agent is ready for, with evidence from Phase 9's
reliability numbers and this topic's attack results.
**Explain:** write the decision and the conditions required to advance a rung. Be honest.

---

# Done when you can answer

1. What three ingredients make an agent hazardous?
2. What does "act as the user" change about impact?
3. Why is egress control so valuable?
4. Where must credentials be attached, and why never in context?
5. Why is identity-from-arguments a vulnerability?
6. What makes an approval prompt genuinely useful?
7. What must an audit trail be able to answer?

Write answers in `notes.md`.
