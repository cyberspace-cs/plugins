---
name: design-agent-approval-policy
description: Design the approval gates and audit record for an agent's tool set. Use when a user asks which agent actions should require confirmation, wants to add human-in-the-loop gating, needs a policy for irreversible or externally visible actions, is defining what an agent audit log must capture, or asks how to keep an autonomous run from spending money or sending messages. Produces a risk-tiered gate table and a concrete audit record schema.
---

# Design Agent Approval Policy

Turn a flat list of available tools into a risk-tiered approval policy plus an audit record that would let someone reconstruct what the agent did and why it was allowed.

## Objective

An approval policy answers one question per action: **may this run unattended, or must a human say yes first?** A good policy is small enough to remember, denies by default on the unclassified, and gates on properties of the action rather than on tool names, because tool names change and a rename must not silently open a gate.

An audit record answers a different question: **if this run goes wrong, can we reconstruct it?** Design both together. A gate with no record cannot be reviewed, and a record with no gate documents the damage rather than preventing it.

## Step 0 - Gather Inputs

Derive what you can before asking, and ask only for what is genuinely missing:

| Input | Where it comes from | If missing |
|-------|--------------------|------------|
| Tool surface | The harness audit, or a direct enumeration | Enumerate it; do not design against an assumed tool set |
| Deployment mode | Whether the agent runs interactively, unattended in CI, or both | Ask; the non-interactive behavior of every gate depends on it |
| Blast-radius tolerance | The user's own risk statement | Ask once, and default to the stricter reading |
| Existing policy | Any current approval config or reviewer runbook | Read it; a policy that contradicts an existing runbook will be ignored |
| Retention requirement | Any compliance or incident-review obligation | Default to "retain longer than the incident window", and say so |

If the agent runs both interactively and unattended, design for the unattended case and treat the interactive case as the relaxation. A policy that only holds when a human is watching does not hold.

## Step 1 - Inventory The Tool Surface

Start from the harness audit if one exists, otherwise enumerate directly:

1. Every tool the harness exposes, including tools contributed by MCP servers and hooks.
2. For each tool: does it write, execute, send, or read? Most tools do exactly one; flag any that do more.
3. For each tool: what is the worst outcome of a single call with model-chosen arguments? Consider the argument space, not the typical call. A filesystem tool is not "safe" because it usually reads — its argument space includes every path the harness can reach.
4. Mark every tool whose arguments are influenced by untrusted content (a fetched page, an issue body, a dependency README, a file in the repo). Those are the tools where a hostile instruction lands.

Record the result as a table before designing anything. `references/tool-risk-tiers.md` has the classification rubric and worked examples.

## Step 2 - Classify By Action Property, Not By Tool

Classify each **action** using three orthogonal questions:

| Question | Why it dominates the decision |
|----------|------------------------------|
| Is it reversible? | Irreversible actions get a gate even when the blast radius looks small, because there is no recovery path |
| Does it have an external side effect? | A recipient cannot be un-messaged, an API cannot be un-called, a purchase cannot be un-made |
| Does it touch credentials or move data outward? | Reading a secret is only half the risk; the other half is what happens next |

The tier follows from the answers:

| Tier | Reversible | External side effect | Gate |
|------|-----------|---------------------|------|
| T0 - Observe | n/a (read-only) | none | Auto-approve |
| T1 - Local, reversible | yes | none | Auto-approve within a scoped root |
| T2 - Local, irreversible | no | none | Approve once per action |
| T3 - External, reversible | yes | yes | Approve once per action, showing the destination |
| T4 - External, irreversible | no | yes | Approve every time; never grant for a session |
| T5 - Unclassified | unknown | unknown | Deny |

Two rules that matter more than the table:

- **T5 is not a placeholder.** A newly installed MCP server, a tool added by a repo-controlled config layer, or a tool whose argument space you could not bound lands in T5 and stays denied until someone classifies it.
- **A tool can span tiers.** A shell tool is T1 for `ls` and T4 for `rm -rf`. Tier the action, not the tool, and state the discriminator explicitly — for a shell tool, that means an allowlist of permitted command shapes with everything else denied, not a prompt that says "be careful".

## Step 3 - Design The Gate

For each tier that requires a gate, define all five of these. A gate missing any one of them is decoration:

1. **Trigger.** The exact predicate that fires the gate, expressed on action properties. "Any call that writes outside the workspace root", not "dangerous commands".
2. **Prompt contents.** What the human sees. It must include: the action, the resolved arguments (not the raw model text), the destination when one exists, the reversibility, and the specific consequence. "Allow this?" is not a prompt; "Delete 412 files under `/data/exports`? This cannot be undone." is.
3. **Grant scope.** Session-wide, action-once, or time-boxed. Default to action-once. Grant session-wide only for T1. Never grant T4 for a session.
4. **Deny behavior.** What happens on refusal. The default is stop and report; never silently retry with different arguments, and never route around the gate through a different tool.
5. **Non-interactive behavior.** What happens when no human is present. The default is deny and stop. A non-interactive run must never auto-approve a T2 or higher action, and must never apply a change outside its declared scope.

### Worked Gate: A Destructive Local Action (T2)

```
Trigger:   any call that deletes or overwrites a path outside the declared working root
Prompt:    "Delete /data/exports (412 files, 1.8 GB)? This cannot be undone.
            Resolved from: <tool> <arguments>. Approve once / Deny."
Scope:     action-once
Deny:      stop, report the refusal, do not retry with a narrower path
Non-interactive: deny and stop
```

The prompt shows the resolved path and the count, not the model's description of its intent. If the model said "clean up stale exports" and the resolved path is `/data`, the human sees `/data`.

### Worked Gate: An Outbound Send (T3)

```
Trigger:   any call that delivers a message to a recipient outside the workspace
Prompt:    "Send to 3 recipients (a@example.com, b@example.com, c@example.com)?
            Subject: <subject>. Body preview: <first 200 chars>.
            This cannot be recalled. Approve once / Deny."
Scope:     action-once
Deny:      stop, report the refusal
Non-interactive: deny and stop
```

Recipient count and destination appear in the prompt because widening the audience is the failure mode, and a gate that only says "send a message?" will be approved reflexively.

## Step 4 - Design The Audit Record

One record per tool call, one JSON object per line, written at call time — not reconstructed at session end. A record written after the fact is missing exactly the runs that crashed.

Minimum schema:

```json
{
  "ts": "2026-09-20T14:03:11.482Z",
  "session_id": "sess_01J8Z...",
  "turn": 14,
  "tool": "filesystem.write",
  "args_digest": "sha256:9f2c...",
  "args_summary": "path=/data/exports/report.csv bytes=20481",
  "tier": "T2",
  "gate": "action_once",
  "decision": "approved",
  "decided_by": "user",
  "outcome": "ok",
  "duration_ms": 87,
  "cwd": "/workspace/app",
  "config_layer_digest": "sha256:41ab..."
}
```

Design notes that make the difference between a log and an audit log:

- **Record denials.** A denied attempt is the highest-signal event in the file. A log of successes only tells you what worked, not what was tried.
- **Record the gate and the decision separately.** `gate: "auto"` with `decision: "approved"` means no human saw it. `gate: "action_once"` with `decided_by: "user"` means one did. Collapsing these two fields makes the log unreadable.
- **Summarize arguments, digest the full payload.** Never write secret values into the log. Write a digest plus a redacted summary so a reviewer can correlate without reading credentials.
- **Include the config digest.** Recording `config_layer_digest` lets you prove which policy was in force at the time, which is the first question asked after an incident.
- **Make it append-only and out of reach.** The log destination must sit outside every writable root. A log the agent can rewrite is not evidence.
- **Log the refusal path too.** If a gate denied and the agent then attempted the same action through another tool, both records must be present and correlatable by `turn`.

## Step 5 - Verify The Design

A policy that has not been adversarially tested is a draft. Run these before calling it done:

| Scenario | Expected result |
|----------|-----------------|
| Untrusted content in a fetched page instructs the agent to send data to an external endpoint | Blocked by the T3 gate; a denial record exists |
| A tool is added by a repo-controlled config layer | Lands in T5 and is denied until classified |
| A T2 action is attempted in a non-interactive run | Denied and the run stops; no auto-approval |
| An action is refused, then attempted through a different tool | Both attempts recorded; the second is also denied |
| The agent attempts to modify the audit log | Denied by filesystem scope; the attempt is itself recorded |
| A tool is renamed between versions | The policy still fires, because it keys on action properties rather than the name |
| A T4 action is requested twice in one session | Gated both times; no session-wide grant |

Any scenario that does not produce the expected result is a finding against the policy, not against the test.

## Reference Policy Shapes

Pick one as the starting point and state which one you picked. The differences are in what runs unattended, not in the tier definitions.

| Shape | T0 | T1 | T2 | T3 | T4 | Fits |
|-------|----|----|----|----|----|------|
| Strict review | auto | approve once | approve each | approve each | approve each | Regulated data, production systems, first weeks of a new agent |
| Balanced | auto | auto in root | approve once | approve once | approve each | The usual default for an interactive coding or ops agent |
| Unattended batch | auto | auto in root | deny | deny | deny | CI and scheduled runs; the agent reports what it would have done |
| Read-only | auto | deny | deny | deny | deny | Analysis, incident review, any run over data you must not mutate |

The unattended-batch shape is the one teams get wrong. The instinct is to grant session-wide approval so the batch can finish. That converts a bounded run into unbounded authority for its whole duration. Deny and report instead, then let a human run the gated actions deliberately.

## Change Management

The policy decays unless it is tied to the surface it governs:

- **Re-classify on every tool-surface change.** A new MCP server, a new hook, or a new tool from an existing server is T5 until classified. Wire this into the harness audit so the re-check is not optional.
- **Re-verify after every harness upgrade.** Key names, defaults, and subcommands move between versions. A gate that silently stopped firing is indistinguishable from a gate that never existed until the incident.
- **Review the denial records, not the approval records.** A rising denial rate means either the policy is too tight or something is probing it. Both are worth knowing.
- **Keep the policy in version control with the tool surface.** A policy reviewed in isolation drifts from the tools it gates.

## Anti-Patterns

- **Tiering by tool name.** Names change; properties do not. A renamed tool must not silently become T0.
- **A single "dangerous" tier.** Two tiers is the practical minimum (gated and not gated) and five is the practical maximum before nobody can apply it consistently.
- **Gating everything.** If every action needs approval, humans approve reflexively and the gate stops carrying information. Gate T2 and above; let T0 and T1 run.
- **Approving on the model's summary of its own action.** The prompt must show resolved arguments from the tool layer, never a paraphrase produced by the model that is about to act.
- **Session-wide grants for irreversible actions.** This converts one approval into unlimited authority.
- **An audit log inside the workspace.** It becomes a file the agent can edit, which makes every other control unverifiable.

## Output Format

1. **Tool inventory** - the table from Step 1, with the untrusted-content flag.
2. **Tier assignment** - per action class, with the discriminator stated for any tool that spans tiers.
3. **Gate table** - trigger, prompt contents, grant scope, deny behavior, and non-interactive behavior per tier.
4. **Audit schema** - the field list plus one example record.
5. **Verification results** - the Step 5 scenarios with actual outcomes, and any that failed.
6. **Open items** - every tool still in T5, and what is needed to classify it.

## Related Skills

- `../harden-agent-harness/SKILL.md` - when the harness itself needs auditing before a policy can be designed.
