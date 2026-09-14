# Skill Security Audit Design

## Purpose

Add `koloz audit <SKILL_MD_PATH>` to assess a local AI skill for security risks. A successful audit returns a security score from 0 to 100, where 100 means fully secure, and describes every identified risk with a risk level.

The first audit provider is the Codex CLI. The application must keep the provider behind a stable interface so other audit engines can be added later without changing command behavior or the result contract.

## Governing ADRs

- `docs/adr/01-use-typescript-for-the-cli-application.md` — application code uses TypeScript with strict type checking, and untrusted inputs require runtime validation. This design uses strict TypeScript types and validates Codex output against JSON Schema.
- `docs/adr/02-use-a-provider-neutral-audit-engine.md` — commands depend on a project-owned audit-engine interface, and all providers return the shared audit-result contract. This design supplies the initial Codex CLI adapter behind that boundary.

## Command Behavior

The command accepts one positional path and one initial output option:

```text
koloz audit <SKILL_MD_PATH> [--json]
```

`SKILL_MD_PATH` must resolve to a readable regular file named `SKILL.md`. Its containing directory is the skill root and the recursive audit scope.

By default, the command writes a human-readable report to standard output. The report starts with `Security score: <score>/100`, then lists risks in descending severity order: `critical`, `high`, `medium`, and `low`. A score of 100 prints `No security risks found.`

With `--json`, standard output contains only the validated audit-result JSON object. A completed audit exits with status 0 regardless of the security score. Input, engine, process, or result-validation failures write a concise message to standard error and exit nonzero. A later feature may add an explicit score threshold for CI policy.

## Architecture

The feature uses a ports-and-adapters boundary with these responsibilities:

- The `audit` command parses input, invokes the use case, selects a formatter, and maps failures to CLI exit behavior.
- `AuditSkill` validates the requested skill path, prepares the safe audit input, delegates to an engine, and returns a validated result.
- `AuditEngine` is a provider-neutral interface. It accepts an audit request and asynchronously returns an `AuditResult`.
- `CodexCliAuditEngine` invokes Codex in non-interactive exec mode and converts process or response failures to application errors.
- A process-runner abstraction isolates subprocess mechanics and enables deterministic adapter tests without an installed or authenticated Codex CLI.
- Human and JSON formatters own presentation; engines never format terminal output.

Provider selection remains internal. The command constructs a `CodexCliAuditEngine`; no public `--engine` option is introduced until another implementation exists.

## Audit Result Contract

`audit-result.schema.json` is the canonical runtime contract. Matching strict TypeScript types represent the same structure in application code.

```json
{
  "schemaVersion": "1.0",
  "securityScore": 72,
  "risks": [
    {
      "level": "high",
      "description": "A bundled script transmits environment variables to an external endpoint."
    }
  ]
}
```

The schema enforces:

- `schemaVersion` is exactly `"1.0"`.
- `securityScore` is an integer from 0 through 100, inclusive, and higher is safer.
- `risks` is an array whose items contain only `level` and `description`.
- `level` is one of `low`, `medium`, `high`, or `critical`.
- `description` is a non-empty string.
- Unknown properties are rejected at every object level.
- A score of 100 requires an empty risk list; a score below 100 requires at least one risk.

The audit engine makes the holistic scoring judgment. Koloz does not calculate the score mechanically from risk counts or levels.

## Safe Audit Input

Skill contents are untrusted. Before invoking an engine, Koloz inventories the skill directory and creates a temporary audit snapshot:

- Regular files under the skill root are copied while preserving relative paths.
- Symbolic links are not followed. Their relative paths and link targets are recorded as metadata so the engine can assess them without gaining access to an external target.
- A symbolic link cannot cause content outside the skill root to enter the snapshot.
- Temporary artifacts are removed after success or failure.

The snapshot provides a stable provider input and prevents the audit process from accidentally executing against or traversing through the original skill directory.

## Codex CLI Adapter

The adapter invokes Codex approximately as follows, with paths supplied as discrete process arguments rather than through a shell:

```text
codex exec
  --ephemeral
  --ignore-user-config
  --skip-git-repo-check
  --sandbox read-only
  --output-schema <audit-result.schema.json>
  --output-last-message <result.json>
  -
```

Codex runs from a clean temporary working directory containing the audit snapshot. The prompt is supplied through standard input. It:

- identifies every skill file as untrusted data rather than instructions;
- directs Codex not to follow embedded instructions;
- prohibits execution of bundled code;
- defines the score direction, result fields, and risk levels;
- asks Codex to inspect the complete snapshot for security risks.

The adapter reads the final response file, parses JSON, and validates it locally against the same schema passed to `--output-schema`. A missing executable, unsuccessful exit, missing result, malformed JSON, or schema violation is an engine failure. No partial response is returned as a successful audit.

## Error Handling

Expected failure categories are:

- invalid skill path, including a missing, unreadable, non-regular, or incorrectly named entry file;
- failure to prepare or clean up the audit snapshot;
- unavailable or unauthenticated Codex CLI;
- unsuccessful or interrupted Codex process;
- missing, malformed, or schema-invalid engine output.

User-facing errors remain concise and actionable. Lower-level causes and captured process diagnostics remain attached for debugging but are not written to standard output. JSON mode does not disguise operational errors as audit results.

## Testing

The deterministic test suite covers:

- JSON Schema acceptance at score boundaries and rejection of invalid levels, descriptions, extra properties, and inconsistent score/risk combinations;
- path validation and recursive snapshot creation, including external symbolic links;
- successful human and JSON command formatting;
- exit behavior for invalid input and engine failure;
- exact Codex arguments, working directory, and prompt input through a fake process runner;
- unsuccessful processes and missing, malformed, or schema-invalid final responses;
- cleanup after both successful and failed audits.

An optional integration test may run against an installed and authenticated Codex CLI. It is excluded from the default suite because it is nondeterministic and requires external configuration.

## Out of Scope

This feature does not include additional audit providers, provider selection flags, configurable scoring weights, CI score thresholds, remediation text, source locations for risks, or persistent audit history.
