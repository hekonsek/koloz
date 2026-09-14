# Use a Provider-Neutral Audit Engine

## Context

Koloz needs to audit local AI skills through the Codex CLI initially. Other model providers or local analyzers may perform the same job later. Command parsing, output formatting, and the public audit-result format should not depend on Codex process arguments or response mechanics.

The application also needs one runtime-validated result contract so every provider reports scores and risks consistently. Provider output is untrusted even when compile-time TypeScript types exist.

## Decision

We will define a project-owned `AuditEngine` interface that accepts an audit request and returns the shared `AuditResult`. CLI commands and application use cases will depend on this interface rather than a provider SDK or executable.

We will define the audit result in a canonical JSON Schema and maintain matching strict TypeScript types. Every engine implementation must return data that passes runtime validation against this schema before the application formats or exposes it.

The first adapter will implement `AuditEngine` by invoking `codex exec`. Codex-specific prompting, process arguments, temporary result handling, and error translation will remain inside that adapter.

## Consequences

Positive consequences:

- New providers can be added without changing the audit command or its output contract.
- Codex-specific details remain isolated and can be tested through a fake process runner.
- All providers expose the same score direction, risk levels, and validation guarantees.
- Command presentation and provider execution can evolve independently.

Negative consequences:

- The interface, adapter, shared contract, and validation boundary introduce more structure than a direct subprocess call.
- Provider-specific capabilities cannot enter the shared result without deliberately evolving the contract.
- The JSON Schema and TypeScript representation must be kept synchronized and tested.
- Different providers may interpret a holistic security score differently even when their output shapes match.

## Alternatives Considered

**Call Codex directly from the command**. This would minimize the initial file count, but it would couple command behavior, subprocess mechanics, prompting, validation, and formatting. Adding another provider would require restructuring the feature or duplicating command logic.

**Configure a generic external process**. Executable, argument, prompt, and parser configuration could support multiple tools, but provider-specific semantics would leak into configuration and weaken the domain-level contract.

**Define a standalone provider plugin protocol**. A stdin/stdout JSON protocol would provide stronger process isolation, but it would require discovery, versioning, packaging, and compatibility mechanisms before a second provider exists.
