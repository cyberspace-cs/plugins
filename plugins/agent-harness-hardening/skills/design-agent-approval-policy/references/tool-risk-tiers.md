# Tool Risk Tiers

Classification rubric and worked examples for `design-agent-approval-policy`.

## Classification Rubric

Answer in this order. The first question that resolves the tier wins, because the questions are ordered by how little recovery they leave.

1. **Can the effect be undone?** If no, the floor is T2. Nothing about a small blast radius changes this — an irreversible action with a small blast radius is still irreversible.
2. **Does anyone outside this machine observe the effect?** If yes, the floor is T3. A recipient, a third-party API, or a public artifact is a side effect you cannot recall.
3. **Does the action touch credentials, or move data toward an external boundary?** If yes, raise one tier, because the read is only half the risk and the egress is the other half.
4. **Can you bound the argument space?** If no, the action is T5 regardless of the answers above. An unbounded argument space is an unclassified capability.
5. **Otherwise**, the action is T0 if it is read-only and T1 if it writes within a scoped root.

## Tier Table

| Tier | Name | Reversible | External effect | Default gate |
|------|------|-----------|-----------------|--------------|
| T0 | Observe | n/a | none | Auto-approve |
| T1 | Local, reversible | yes | none | Auto-approve within a scoped root |
| T2 | Local, irreversible | no | none | Approve once per action |
| T3 | External, reversible | yes | yes | Approve once per action, showing the destination |
| T4 | External, irreversible | no | yes | Approve every time; never grant for a session |
| T5 | Unclassified | unknown | unknown | Deny |

## Worked Examples

### A filesystem tool

Not one tier. Split it by the argument:

| Action | Tier | Discriminator |
|--------|------|---------------|
| Read a file inside the workspace root | T0 | Read-only, path inside root |
| Write or edit a file inside the workspace root | T1 | Path inside root, and the path is not a credential path or `.git` |
| Write a file outside the workspace root | T2 | Path outside root; overwriting is irreversible |
| Delete a path recursively | T2 | Irreversible; gate even inside the root |
| Read a credential path | T4 | Credential access; the read itself is not recoverable once the value is in context |

The last row is the one teams miss. Reading a secret is not reversible in any meaningful sense: once the value is in the model's context, it can leave through any other channel, including one that is otherwise T1.

### A shell or command tool

Unbounded by default, therefore T5 until an allowlist narrows it. Then:

| Action | Tier | Discriminator |
|--------|------|---------------|
| An allowlisted read-only command shape | T0 | Matches the allowlist exactly |
| An allowlisted command that writes inside the root | T1 | Matches the allowlist and stays inside root |
| Any command shape not on the allowlist | T5 | Unbounded argument space |

An allowlist of permitted command shapes is the only construction that bounds a shell tool. A prompt that says "be careful with destructive commands" does not bound it, because the prompt is advisory and the shell is not.

### A message or mail tool

| Action | Tier | Discriminator |
|--------|------|---------------|
| Read the inbox | T0 | Read-only |
| Draft a message without sending | T1 | Local, reversible |
| Send to an internal recipient | T3 | External, reversible only in the sense that a correction can follow |
| Send to an external recipient, or send to a distribution list | T4 | Cannot be recalled; distribution widens blast radius without changing reversibility |

Sending is the canonical T4 despite looking harmless. It cannot be undone, it crosses an external boundary, and it is the most common exfiltration channel for an agent that has already read something it should not have.

### A payment or spend tool

Always T4. The action crosses an external boundary and money is not recoverable. Gate every call, never grant for a session, and require the amount and destination to appear in the approval prompt.

### An HTTP fetch or browser tool

| Action | Tier | Discriminator |
|--------|------|---------------|
| Fetch a URL on an allowlist | T0 | Read-only, allowlisted destination |
| Fetch an arbitrary URL | T3 | Egress to an unvetted destination; the response is also untrusted input to every other tool |
| Submit a form or click a control that mutates remote state | T3 or T4 | T3 when the remote state is recoverable, T4 when it is not |

An arbitrary-URL fetch is a two-sided risk. It is egress, and it is also the primary delivery mechanism for instructions that will drive every other tool in the session.

### A package or dependency tool

| Action | Tier | Discriminator |
|--------|------|---------------|
| Install from a lockfile with a verified integrity hash | T1 | Local, reversible, integrity-pinned |
| Install an unpinned or newly published version | T4 | Installs and then executes arbitrary code from a third party |
| Publish a package | T4 | External, irreversible, and version numbers cannot be reclaimed |

Unpinned installation is T4 because it is code execution with an external supply chain, even though the tool's own name suggests a local file operation.

### An MCP server added by a repository

T5, unconditionally, until a human classifies it. The repository is under someone else's control, and the server arrives with whatever tools it declares. This is the single most important T5 case: it is the path by which an outside contributor adds capability to a run they do not otherwise control.

## Review Checklist For A Tier Assignment

- [ ] The tier is assigned to an action class, not to a tool name.
- [ ] Every tool that spans tiers has its discriminator written down explicitly.
- [ ] Every action with an unbounded argument space is T5.
- [ ] Credential reads are T4 regardless of how local they look.
- [ ] Sends and publishes are T4 regardless of how small the audience looks.
- [ ] Every T5 item names what is required to classify it.
- [ ] Renaming a tool would not change any tier in the table.
