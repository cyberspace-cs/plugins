# Permission Boundary Checklist

Probe list for `harden-agent-harness`. Work top to bottom. Every line is a question with an evidence source, not an assertion — the installed CLI and the resolved config layers are the source of truth.

## How To Use This Checklist

1. Resolve the layer stack first (`harden-agent-harness` Step 1). A probe answered against the wrong layer is worse than an unanswered probe.
2. For each item, record one of: `pass`, `fail`, `not-applicable`, or `unknown`. Never record `pass` from inference alone.
3. An `unknown` on a `critical` or `high` item is itself a finding. Report it as such rather than leaving it blank.
4. Re-run the full list after a harness upgrade. Defaults, key names, and subcommands move between versions.

## S1 - Execution Sandbox

- [ ] The installed CLI version is recorded (`codex --version`).
- [ ] Every layer that sets an approval-related key has been read, not just the user layer.
- [ ] The **effective** approval value is recorded, along with the layer that supplies it.
- [ ] Every layer that sets a sandbox-related key has been read, and the effective value is recorded.
- [ ] The names of the approval and sandbox keys used in the report were confirmed against `codex --help` output for this version.
- [ ] No bypass flag appears in a shell alias, function, wrapper script, `Makefile`, `Taskfile`, `package.json` script, or CI workflow.
- [ ] No project-level layer raises privilege above the user layer for a project that receives outside contributions.
- [ ] No profile layer silently relaxes the base user configuration.
- [ ] A least-privileged probe was run and the harness denied the action, with the command and exit status recorded as evidence.
- [ ] If confinement is disabled anywhere, the justification is documented and scoped to a single trusted directory rather than set globally.

## S2 - Filesystem Scope

- [ ] Every writable root is enumerated.
- [ ] Every explicit deny rule is enumerated.
- [ ] No credential directory sits inside a writable root.
- [ ] `.git` is not writable inside a scoped workspace.
- [ ] `project_root_markers` is enumerated, and every marker is a file an outside contributor could add to a directory.
- [ ] The set of directories eligible for a repo-controlled config layer is recorded.
- [ ] Symlink handling was considered: a writable root containing a symlink to a denied path is not confined.
- [ ] A read of a sentinel path outside the workspace was attempted and denied, with evidence recorded.

## S3 - Network Egress

- [ ] Whether network access is permitted is recorded, and the answer distinguishes "disabled" from "enabled with an empty allowlist".
- [ ] If an allowlist exists, every entry is enumerated and each entry's necessity is stated.
- [ ] No allowlist entry uses a wildcard or a broadly shared CDN or proxy domain.
- [ ] Every HTTP-based MCP server is inventoried as an egress path.
- [ ] No HTTP MCP endpoint is supplied by a repo-controlled config layer.
- [ ] A sandboxed request to an external host was attempted and denied, with evidence recorded.
- [ ] If egress is permitted, every installed tool that can send data to an arbitrary destination is listed as a compensating risk.

## S4 - Credential Reach

- [ ] Each of these was checked for existence, path, and mode, without printing contents:

  - [ ] `$CODEX_HOME`
  - [ ] `~/.ssh`
  - [ ] `~/.aws`
  - [ ] `~/.config/gh`
  - [ ] `~/.config/gcloud`
  - [ ] `~/.kube`
  - [ ] `~/.netrc`
  - [ ] `~/.npmrc`
  - [ ] `~/.docker/config.json`
  - [ ] `~/.git-credentials`
  - [ ] `~/.config/op` and other secrets-manager directories in use
  - [ ] `.env` and `.env.*` inside every readable root

- [ ] No credential path is readable from the shell under the effective filesystem scope.
- [ ] No MCP server declares a filesystem root that covers a credential directory.
- [ ] Any component that hands configuration to repository-influenced commands receives a sanitized copy rather than the credential-bearing home.
- [ ] Long-lived credentials are distinguished from short-lived, scoped credentials, and the report says which one is exposed.
- [ ] A read of one sentinel credential path was attempted and denied, with evidence recorded.

## S5 - Tool And MCP Surface

- [ ] Every MCP server is enumerated across every layer **and** every repo-local `.mcp.json`.
- [ ] For each server: transport is recorded (`stdio` or `http`).
- [ ] For each `stdio` server: the `command` is an absolute path or a pinned package, and it does not resolve inside the repository.
- [ ] For each `stdio` server: every package in `args` carries an explicit version. `npx -y <pkg>` with no version is a finding.
- [ ] For each `http` server: an OAuth resource is declared, or the absence is reported as a finding.
- [ ] For each server: the tool count is recorded, and every tool that writes, executes, or sends is flagged.
- [ ] No server is introduced by a repo-controlled config layer.
- [ ] Hook definitions (`hooks.json` or equivalent) are enumerated, and each hook's command is inspected for repo-relative resolution.
- [ ] The hooks surface is treated as execution, not configuration: a hook that runs a committed script runs contributor-controlled code.

## S6 - Approval And Audit

- [ ] The classes of action requiring approval are enumerated.
- [ ] The classes of auto-approved action are enumerated.
- [ ] Irreversible actions are gated: force push, history rewrite, tag or release deletion, package publish, infrastructure destroy, database migration.
- [ ] External-side-effect actions are gated: sending mail or chat messages, posting to a third-party API, creating or modifying records in a system of record, any action that spends money.
- [ ] The audit destination is identified, or its absence is reported.
- [ ] The audit record captures: timestamp, session identity, tool name, arguments, decision, and outcome.
- [ ] Denied attempts are recorded, not only allowed ones.
- [ ] The audit destination is append-only from the agent's perspective.
- [ ] The audit destination survives session end.
- [ ] The audit destination is outside every writable root.
- [ ] A sample record is inspected to confirm the schema matches the claim, rather than trusting the documentation.

## Reporting Reminders

- Cite `file:line` for every config claim. A claim without a location is not evidence.
- Name the layer that supplies each effective value. "Sandbox is enabled" is incomplete; "sandbox is enabled by the user layer at `$CODEX_HOME/config.toml:12` and not overridden by the trusted project layer" is a finding.
- Mark anything you could not determine as `unknown` and say what would resolve it.
- Never include a credential value, a token prefix, or a key fingerprint in the report.
