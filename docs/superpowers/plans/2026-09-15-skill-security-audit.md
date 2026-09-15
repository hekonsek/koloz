# Skill Security Audit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `koloz audit <SKILL_MD_PATH> [--json]` to safely snapshot a local skill, audit it through Codex CLI, validate the provider-neutral result, and render deterministic human or JSON output.

**Architecture:** Implement the audit feature as a service-oriented hexagon. A Commander input adapter owns argument parsing, output, and exit status; `AuditSkill` owns validation and snapshot lifecycle; a project-owned `AuditEngine` port isolates the Codex CLI output adapter and its safe `execFile` process runner.

**Tech Stack:** Node.js 22+, npm, strict ESM TypeScript, Commander, Ajv, Vitest, and Node built-ins (`fs`, `path`, `os`, `child_process`).

**Spec:** `docs/superpowers/specs/2026-09-14-skill-security-audit-design.md`

## Global Constraints

- Application code uses TypeScript with strict type checking, and untrusted inputs require runtime validation.
- Commands depend on a project-owned audit-engine interface, and all providers return the shared audit-result contract.
- `SKILL_MD_PATH` must resolve to a readable regular file named `SKILL.md`; its containing directory is the recursive audit scope.
- A completed audit exits with status 0 regardless of security score; input, engine, process, or result-validation failures write a concise message to standard error and exit nonzero.
- JSON mode writes only the validated audit-result object to standard output.
- Symbolic links are never followed; only their relative paths and targets enter the audit input as metadata.
- Codex receives untrusted skill contents only through a temporary snapshot, runs non-interactively with a read-only sandbox, and never receives shell-interpolated arguments.
- Temporary artifacts are removed after both successful and failed audits.
- Additional providers, provider flags, score thresholds, remediation text, risk source locations, and persistent history remain out of scope.

## Governing ADRs

- `docs/adr/01-use-typescript-for-the-cli-application.md` — use strict TypeScript and validate untrusted values at runtime. Tasks 1 and 2 establish strict compilation and JSON Schema validation.
- `docs/adr/02-use-a-provider-neutral-audit-engine.md` — keep Codex details behind `AuditEngine` and enforce one shared result contract. Tasks 2, 4, and 5 create and test those boundaries.

## Consulted Guidance

- `skill-node` — compile TypeScript for Node, organize business behavior by service, keep tests under top-level `test`, expose reproducible npm scripts, and preserve error causes at boundaries.
- `skill-node-cli` — use Commander only in the input adapter, keep stdout machine-readable, and use `execFile` with discrete arguments for external programs.
- `skill-superpowers-adr` and `skill-adr` — translate accepted ADR constraints into tests and record the otherwise-unresolved runtime/framework choice before implementation.
- `skill-superpowers-writing-plans-use-skills` — carry the relevant Node, CLI, and ADR guidance into this plan.

## File Structure

```text
docs/adr/
  03-use-nodejs-commander-and-npm-for-the-cli-runtime.md
schema/
  audit-result.schema.json
src/
  index.ts
  adapters/in/cli/
    cli.ts
    format-audit-result.ts
  services/audit/
    audit-engine.ts
    audit-errors.ts
    audit-result.ts
    audit-skill.ts
    audit-snapshot.ts
    validate-audit-result.ts
    adapters/out/codex-cli/
      codex-cli-audit-engine.ts
      codex-audit-prompt.ts
      exec-file-process-runner.ts
      process-runner.ts
test/
  adapters/in/cli/
    cli.test.ts
    format-audit-result.test.ts
  services/audit/
    audit-skill.test.ts
    audit-snapshot.test.ts
    validate-audit-result.test.ts
    adapters/out/codex-cli/
      codex-cli-audit-engine.test.ts
      exec-file-process-runner.test.ts
package.json
package-lock.json
tsconfig.json
vitest.config.ts
vitest.integration.config.ts
```

The schema is stored outside `src` because `tsc` does not copy arbitrary JSON. Both source-time tests and emitted files resolve it from the repository/package root, and the package includes `schema/` alongside `dist/`.

---

### Task 1: Record and Bootstrap the CLI Toolchain

**Files:**
- Create: `docs/adr/03-use-nodejs-commander-and-npm-for-the-cli-runtime.md`
- Create: `package.json`
- Create: `package-lock.json`
- Create: `tsconfig.json`
- Create: `vitest.config.ts`
- Create: `src/index.ts`
- Test: `test/index.test.ts`

**Interfaces:**
- Consumes: accepted ADR 01's strict TypeScript requirement.
- Produces: npm scripts `build`, `typecheck`, and `test`; executable `koloz`; exported `main(argv: readonly string[]): Promise<number>` placeholder seam that Task 7 replaces with CLI wiring.

- [ ] **Step 1: Write the runtime/toolchain ADR**

Create `docs/adr/03-use-nodejs-commander-and-npm-for-the-cli-runtime.md` with the repository's existing ADR sections. Record Node.js 22+ and npm for runtime/package management, Commander for parsing, `tsc` for builds, and Vitest for tests. State the costs: a Node runtime is required, Commander becomes an adapter dependency, and `schema/` must be distributed next to `dist/`. Alternatives must cover Bun, `util.parseArgs`, and a bundled single-file executable.

- [ ] **Step 2: Initialize the package and install dependencies**

Run:

```bash
npm init -y
npm install commander ajv
npm install --save-dev typescript vitest @types/node
```

Expected: `package.json` and a reproducible `package-lock.json` are created.

- [ ] **Step 3: Configure the package scripts and executable**

Edit `package.json` to include these fields while preserving the installed dependency versions:

```json
{
  "type": "module",
  "engines": { "node": ">=22" },
  "bin": { "koloz": "./dist/index.js" },
  "files": ["dist", "schema"],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "vitest run"
  }
}
```

- [ ] **Step 4: Add strict TypeScript and Vitest configuration**

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"]
}
```

Create `vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: { environment: "node", include: ["test/**/*.test.ts"] },
});
```

- [ ] **Step 5: Write the failing executable smoke test**

Create `test/index.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { main } from "../src/index.js";

describe("main", () => {
  it("returns a promise of an exit code", async () => {
    await expect(main(["node", "koloz", "--help"])).resolves.toBe(0);
  });
});
```

- [ ] **Step 6: Run the smoke test to verify it fails**

Run: `npm test -- test/index.test.ts`

Expected: FAIL because `src/index.ts` does not exist.

- [ ] **Step 7: Add the minimal executable seam**

Create `src/index.ts`:

```ts
#!/usr/bin/env node

export async function main(_argv: readonly string[]): Promise<number> {
  return 0;
}

if (import.meta.url === new URL(process.argv[1] ?? "", "file:").href) {
  process.exitCode = await main(process.argv);
}
```

- [ ] **Step 8: Verify bootstrap and commit**

Run:

```bash
npm test -- test/index.test.ts
npm run typecheck
npm run build
```

Expected: all commands exit 0 and `dist/index.js` starts with the Node shebang.

Commit:

```bash
git add docs/adr/03-use-nodejs-commander-and-npm-for-the-cli-runtime.md package.json package-lock.json tsconfig.json vitest.config.ts src/index.ts test/index.test.ts
git commit -m "build: bootstrap TypeScript CLI"
```

---

### Task 2: Define and Runtime-Validate the Audit Result Contract

**Files:**
- Create: `schema/audit-result.schema.json`
- Create: `src/services/audit/audit-result.ts`
- Create: `src/services/audit/validate-audit-result.ts`
- Test: `test/services/audit/validate-audit-result.test.ts`

**Interfaces:**
- Consumes: Ajv from Task 1.
- Produces: `RiskLevel`, `AuditRisk`, `AuditResult`, `severityRank`, and `validateAuditResult(value: unknown): AuditResult`.

- [ ] **Step 1: Add the canonical JSON Schema**

Create `schema/audit-result.schema.json`:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "additionalProperties": false,
  "required": ["schemaVersion", "securityScore", "risks"],
  "properties": {
    "schemaVersion": { "const": "1.0" },
    "securityScore": { "type": "integer", "minimum": 0, "maximum": 100 },
    "risks": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["level", "description"],
        "properties": {
          "level": { "enum": ["low", "medium", "high", "critical"] },
          "description": { "type": "string", "minLength": 1 }
        }
      }
    }
  },
  "allOf": [
    {
      "if": { "properties": { "securityScore": { "const": 100 } } },
      "then": { "properties": { "risks": { "maxItems": 0 } } },
      "else": { "properties": { "risks": { "minItems": 1 } } }
    }
  ]
}
```

- [ ] **Step 2: Define the matching strict TypeScript contract**

Create `src/services/audit/audit-result.ts`:

```ts
export type RiskLevel = "low" | "medium" | "high" | "critical";

export interface AuditRisk {
  readonly level: RiskLevel;
  readonly description: string;
}

export interface AuditResult {
  readonly schemaVersion: "1.0";
  readonly securityScore: number;
  readonly risks: readonly AuditRisk[];
}

export const severityRank: Readonly<Record<RiskLevel, number>> = {
  low: 0,
  medium: 1,
  high: 2,
  critical: 3,
};
```

- [ ] **Step 3: Write failing schema-boundary tests**

Create table-driven tests in `test/services/audit/validate-audit-result.test.ts` that accept scores `0`, `99`, and `100` with their required risk cardinality. Reject a score outside `0..100`, a fractional score, an unknown risk level, an empty description, extra root and risk properties, risks at score 100, no risks below 100, and non-object input. Assert rejection throws `InvalidAuditResultError` and includes Ajv diagnostics in its `cause`.

Use this representative structure:

```ts
expect(validateAuditResult({ schemaVersion: "1.0", securityScore: 100, risks: [] }))
  .toEqual({ schemaVersion: "1.0", securityScore: 100, risks: [] });

expect(() => validateAuditResult({
  schemaVersion: "1.0",
  securityScore: 100,
  risks: [{ level: "low", description: "unexpected" }],
})).toThrow(InvalidAuditResultError);
```

- [ ] **Step 4: Run the validator test to verify it fails**

Run: `npm test -- test/services/audit/validate-audit-result.test.ts`

Expected: FAIL because the validator and error do not exist.

- [ ] **Step 5: Implement the compiled validator**

Create `src/services/audit/validate-audit-result.ts`. Load and compile the schema once:

```ts
import { readFileSync } from "node:fs";
import Ajv from "ajv";
import type { AuditResult } from "./audit-result.js";

const schemaUrl = new URL(
  "../../../schema/audit-result.schema.json",
  import.meta.url,
);
const schema: unknown = JSON.parse(readFileSync(schemaUrl, "utf8"));
const validate = new Ajv({ allErrors: true }).compile(schema);
```

Then expose:

```ts
export class InvalidAuditResultError extends Error {
  constructor(cause: unknown) {
    super("Audit engine returned an invalid result", { cause });
    this.name = "InvalidAuditResultError";
  }
}

export function validateAuditResult(value: unknown): AuditResult {
  if (!validate(value)) {
    throw new InvalidAuditResultError(structuredClone(validate.errors));
  }
  return value as AuditResult;
}
```

The schema-loading error must surface during startup/tests rather than being converted into provider output.

- [ ] **Step 6: Verify the result contract and commit**

Run:

```bash
npm test -- test/services/audit/validate-audit-result.test.ts
npm run typecheck
npm run build
```

Expected: all commands exit 0, and the validator resolves the same root-relative schema from `src` and `dist`.

Commit:

```bash
git add schema/audit-result.schema.json src/services/audit/audit-result.ts src/services/audit/validate-audit-result.ts test/services/audit/validate-audit-result.test.ts
git commit -m "feat: define audit result contract"
```

---

### Task 3: Validate Skill Paths and Create Safe Snapshots

**Files:**
- Create: `src/services/audit/audit-errors.ts`
- Create: `src/services/audit/audit-snapshot.ts`
- Test: `test/services/audit/audit-snapshot.test.ts`

**Interfaces:**
- Consumes: a user-supplied `SKILL.md` path.
- Produces: `AuditSnapshot { workspacePath: string; skillPath: string; symlinkManifestPath: string; symlinks: readonly SymlinkRecord[] }`, `createAuditSnapshot(skillMarkdownPath: string): Promise<AuditSnapshot>`, and `removeAuditSnapshot(snapshot: AuditSnapshot): Promise<void>`.

- [ ] **Step 1: Define typed application errors**

Create `src/services/audit/audit-errors.ts`:

```ts
export abstract class AuditError extends Error {
  abstract readonly publicMessage: string;
}

export class InvalidSkillPathError extends AuditError {
  readonly publicMessage: string;
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
    this.name = "InvalidSkillPathError";
    this.publicMessage = message;
  }
}

export class SnapshotPreparationError extends AuditError {
  readonly publicMessage = "Could not prepare a safe audit snapshot";
  constructor(cause: unknown) {
    super("Could not prepare a safe audit snapshot", { cause });
    this.name = "SnapshotPreparationError";
  }
}

export class SnapshotCleanupError extends AuditError {
  readonly publicMessage = "Could not remove the temporary audit snapshot";
  constructor(cause: unknown) {
    super("Could not remove the temporary audit snapshot", { cause });
    this.name = "SnapshotCleanupError";
  }
}
```

- [ ] **Step 2: Write failing path and snapshot tests**

In `test/services/audit/audit-snapshot.test.ts`, create each fixture under `mkdtemp(join(tmpdir(), "koloz-test-"))` and remove it in `afterEach`. Cover:

- missing input;
- basename other than `SKILL.md`;
- directory or symbolic link passed as the entry file;
- unreadable `SKILL.md` (skip this assertion on platforms where permission bits cannot make a file unreadable);
- recursive copying of regular files with preserved relative paths;
- empty directories being irrelevant to the snapshot;
- internal and external symlinks recorded but not followed;
- manifest JSON containing only sorted `{ path, target }` entries;
- explicit cleanup removing the whole workspace.

Use an external sentinel file and assert its content never appears under `snapshot.skillPath`.

- [ ] **Step 3: Run snapshot tests to verify they fail**

Run: `npm test -- test/services/audit/audit-snapshot.test.ts`

Expected: FAIL because snapshot functions do not exist.

- [ ] **Step 4: Implement safe traversal and copying**

Create `src/services/audit/audit-snapshot.ts` using only promise-based Node filesystem APIs. Validate the entry with `lstat`, require the exact basename, and check readability with `access(path, constants.R_OK)`. Create `mkdtemp(join(tmpdir(), "koloz-audit-"))`, then recursively:

```ts
const entry = await lstat(sourcePath);
if (entry.isSymbolicLink()) {
  symlinks.push({
    path: relative(skillRoot, sourcePath),
    target: await readlink(sourcePath),
  });
  return;
}
if (entry.isDirectory()) {
  await mkdir(destinationPath, { recursive: true });
  for (const name of (await readdir(sourcePath)).sort()) {
    await copyEntry(join(sourcePath, name), join(destinationPath, name));
  }
  return;
}
if (entry.isFile()) {
  await mkdir(dirname(destinationPath), { recursive: true });
  await copyFile(sourcePath, destinationPath, constants.COPYFILE_EXCL);
}
```

Ignore non-regular special files so the audit never opens devices, sockets, or FIFOs. Write sorted link metadata to `<workspace>/symlinks.json`, outside `<workspace>/skill`, and wrap preparation failures while removing any partially-created workspace.

- [ ] **Step 5: Implement explicit cleanup**

Implement `removeAuditSnapshot` with `rm(snapshot.workspacePath, { recursive: true, force: true })`, translating failures to `SnapshotCleanupError` without hiding the original cause.

- [ ] **Step 6: Verify safe snapshot behavior and commit**

Run:

```bash
npm test -- test/services/audit/audit-snapshot.test.ts
npm run typecheck
```

Expected: all assertions pass, including the external-symlink sentinel check.

Commit:

```bash
git add src/services/audit/audit-errors.ts src/services/audit/audit-snapshot.ts test/services/audit/audit-snapshot.test.ts
git commit -m "feat: create safe skill audit snapshots"
```

---

### Task 4: Add the Provider-Neutral Engine and Safe Process Runner

**Files:**
- Create: `src/services/audit/audit-engine.ts`
- Create: `src/services/audit/adapters/out/codex-cli/process-runner.ts`
- Create: `src/services/audit/adapters/out/codex-cli/exec-file-process-runner.ts`
- Test: `test/services/audit/adapters/out/codex-cli/exec-file-process-runner.test.ts`

**Interfaces:**
- Consumes: `AuditSnapshot` from Task 3.
- Produces: `AuditEngine.audit(request: AuditRequest): Promise<AuditResult>` and `ProcessRunner.run(request: ProcessRequest): Promise<ProcessResult>`.

- [ ] **Step 1: Define provider-neutral engine types**

Create `src/services/audit/audit-engine.ts`:

```ts
import type { AuditResult } from "./audit-result.js";
import type { SymlinkRecord } from "./audit-snapshot.js";

export interface AuditRequest {
  readonly workspacePath: string;
  readonly skillPath: string;
  readonly symlinkManifestPath: string;
  readonly symlinks: readonly SymlinkRecord[];
  readonly signal?: AbortSignal;
}

export interface AuditEngine {
  audit(request: AuditRequest): Promise<AuditResult>;
}
```

- [ ] **Step 2: Define process runner types**

Create `process-runner.ts`:

```ts
export interface ProcessRequest {
  readonly executable: string;
  readonly args: readonly string[];
  readonly cwd: string;
  readonly stdin: string;
  readonly signal?: AbortSignal;
}

export interface ProcessResult {
  readonly exitCode: number;
  readonly stdout: string;
  readonly stderr: string;
}

export interface ProcessRunner {
  run(request: ProcessRequest): Promise<ProcessResult>;
}
```

- [ ] **Step 3: Write failing runner tests**

Test `ExecFileProcessRunner` with a tiny Node child process invoked through `process.execPath`. Assert it:

- passes stdin literally, including `$(touch should-not-exist)` and `;` characters;
- uses the requested working directory;
- resolves stdout, stderr, and exit code for success and nonzero exit;
- rejects executable-not-found and aborted execution with the original cause;
- never enables a shell.

- [ ] **Step 4: Run runner tests to verify they fail**

Run: `npm test -- test/services/audit/adapters/out/codex-cli/exec-file-process-runner.test.ts`

Expected: FAIL because `ExecFileProcessRunner` does not exist.

- [ ] **Step 5: Implement `ExecFileProcessRunner`**

Wrap `child_process.execFile` in a Promise. Pass `cwd`, `signal`, UTF-8 encoding, and a fixed 10 MiB `maxBuffer`; never pass `shell`. Write the exact prompt to `child.stdin`, then end stdin. Resolve numeric nonzero exit codes as `ProcessResult`; reject spawn/configuration/abort errors whose `code` is not numeric. Preserve captured stdout/stderr on the resolved result.

- [ ] **Step 6: Verify the process boundary and commit**

Run:

```bash
npm test -- test/services/audit/adapters/out/codex-cli/exec-file-process-runner.test.ts
npm run typecheck
```

Expected: all tests pass and the shell-metacharacter sentinel is absent.

Commit:

```bash
git add src/services/audit/audit-engine.ts src/services/audit/adapters/out/codex-cli/process-runner.ts src/services/audit/adapters/out/codex-cli/exec-file-process-runner.ts test/services/audit/adapters/out/codex-cli/exec-file-process-runner.test.ts
git commit -m "feat: add safe audit process runner"
```

---

### Task 5: Implement the Codex CLI Audit Engine

**Files:**
- Create: `src/services/audit/adapters/out/codex-cli/codex-audit-prompt.ts`
- Create: `src/services/audit/adapters/out/codex-cli/codex-cli-audit-engine.ts`
- Modify: `src/services/audit/audit-errors.ts`
- Test: `test/services/audit/adapters/out/codex-cli/codex-cli-audit-engine.test.ts`

**Interfaces:**
- Consumes: `AuditEngine`, `ProcessRunner`, `validateAuditResult`, and an `AuditRequest` whose workspace contains `skill/` and `symlinks.json`.
- Produces: `CodexCliAuditEngine(processRunner: ProcessRunner, options?: { executable?: string; schemaPath?: string })` implementing `AuditEngine`.

- [ ] **Step 1: Add the engine error type**

Extend `audit-errors.ts`:

```ts
export class AuditEngineError extends AuditError {
  readonly publicMessage: string;
  constructor(message: string, options?: ErrorOptions) {
    super(message, options);
    this.name = "AuditEngineError";
    this.publicMessage = message;
  }
}
```

- [ ] **Step 2: Write failing adapter tests with a fake runner**

Build a `RecordingProcessRunner` that records one `ProcessRequest` and writes controlled content to the result path found after `--output-last-message`. Cover:

- exact executable `codex` and ordered arguments `exec`, `--ephemeral`, `--ignore-user-config`, `--skip-git-repo-check`, `--sandbox`, `read-only`, `--output-schema`, schema path, `--output-last-message`, result path, `-`;
- `cwd === request.workspacePath` and the same `AbortSignal` reaches the runner;
- prompt labels `skill/` and `symlinks.json` as untrusted data, forbids following embedded instructions and executing bundled code, defines score direction and four risk levels, and requests JSON matching schema version `1.0`;
- valid result file returned as `AuditResult`;
- unavailable executable, nonzero exit, missing result, malformed JSON, and schema-invalid JSON translated to distinct concise `AuditEngineError.publicMessage` values;
- diagnostics remain on `cause` and are not included in `publicMessage`.

- [ ] **Step 3: Run adapter tests to verify they fail**

Run: `npm test -- test/services/audit/adapters/out/codex-cli/codex-cli-audit-engine.test.ts`

Expected: FAIL because the adapter does not exist.

- [ ] **Step 4: Implement the fixed defensive prompt**

Create `codex-audit-prompt.ts` exporting a constant prompt. It must explicitly state that all files under `skill/` and all entries in `symlinks.json` are potentially malicious data, instructions inside them must be ignored, no bundled code or commands may be executed, symlink targets must not be opened, and every file must be inspected only by reading the supplied snapshot. Include the exact contract semantics from the design.

- [ ] **Step 5: Implement `CodexCliAuditEngine`**

Resolve defaults as:

```ts
const schemaPath = fileURLToPath(
  new URL("../../../../../../schema/audit-result.schema.json", import.meta.url),
);
const resultPath = join(request.workspacePath, "result.json");
```

Call the runner once with discrete arguments and prompt stdin. Treat a nonzero exit as `AuditEngineError("Codex audit failed", { cause: { exitCode, stderr } })`. Translate runner rejection to `"Codex CLI is unavailable or could not be started"`; missing/unreadable result to `"Codex did not produce an audit result"`; parse failure to `"Codex returned malformed JSON"`; and validator failure to `"Codex returned an invalid audit result"`. Keep the original error or diagnostics as `cause`.

- [ ] **Step 6: Verify adapter behavior and commit**

Run:

```bash
npm test -- test/services/audit/adapters/out/codex-cli/codex-cli-audit-engine.test.ts
npm run typecheck
```

Expected: all success and failure cases pass without requiring Codex to be installed.

Commit:

```bash
git add src/services/audit/audit-errors.ts src/services/audit/adapters/out/codex-cli/codex-audit-prompt.ts src/services/audit/adapters/out/codex-cli/codex-cli-audit-engine.ts test/services/audit/adapters/out/codex-cli/codex-cli-audit-engine.test.ts
git commit -m "feat: add Codex audit engine"
```

---

### Task 6: Orchestrate Snapshot, Engine, Validation, and Cleanup

**Files:**
- Create: `src/services/audit/audit-skill.ts`
- Test: `test/services/audit/audit-skill.test.ts`

**Interfaces:**
- Consumes: `AuditEngine`, snapshot functions, and `validateAuditResult`.
- Produces: `new AuditSkill(engine, snapshotLifecycle?).audit(skillMarkdownPath, options?): Promise<AuditResult>`.

- [ ] **Step 1: Write failing orchestration tests**

Inject a snapshot lifecycle with `create` and `remove` spies and a fake `AuditEngine`. Assert:

- the engine receives paths and symlink metadata from the created snapshot;
- `signal` reaches the engine;
- valid results are returned;
- an invalid fake-engine result is rejected at the use-case boundary;
- cleanup runs once after success, engine failure, and result-validation failure;
- cleanup failure after success surfaces `SnapshotCleanupError`;
- simultaneous engine and cleanup failures preserve both through `AggregateError`, wrapped by `SnapshotCleanupError`.

- [ ] **Step 2: Run orchestration tests to verify they fail**

Run: `npm test -- test/services/audit/audit-skill.test.ts`

Expected: FAIL because `AuditSkill` does not exist.

- [ ] **Step 3: Implement `AuditSkill`**

Use this public API:

```ts
export interface AuditSkillOptions {
  readonly signal?: AbortSignal;
}

export interface SnapshotLifecycle {
  create(path: string): Promise<AuditSnapshot>;
  remove(snapshot: AuditSnapshot): Promise<void>;
}

export class AuditSkill {
  constructor(
    private readonly engine: AuditEngine,
    private readonly snapshots: SnapshotLifecycle = defaultSnapshotLifecycle,
  ) {}

  async audit(path: string, options: AuditSkillOptions = {}): Promise<AuditResult> {
    const snapshot = await this.snapshots.create(path);
    let primaryFailure: unknown;
    try {
      return validateAuditResult(await this.engine.audit({
        ...snapshot,
        ...(options.signal === undefined ? {} : { signal: options.signal }),
      }));
    } catch (error) {
      primaryFailure = error;
      throw error;
    } finally {
      try {
        await this.snapshots.remove(snapshot);
      } catch (cleanupFailure) {
        const cleanupError = cleanupFailure instanceof SnapshotCleanupError
          ? cleanupFailure
          : new SnapshotCleanupError(cleanupFailure);
        if (primaryFailure === undefined) {
          throw cleanupError;
        }
        throw new SnapshotCleanupError(
          new AggregateError([primaryFailure, cleanupError]),
        );
      }
    }
  }
}
```

Track the primary failure. In `finally`, if cleanup also fails, throw `SnapshotCleanupError(new AggregateError([primaryFailure, cleanupFailure]))`; if only cleanup fails, throw it directly. Never return until cleanup succeeds.

- [ ] **Step 4: Verify orchestration and commit**

Run:

```bash
npm test -- test/services/audit/audit-skill.test.ts
npm run typecheck
```

Expected: all lifecycle cases pass.

Commit:

```bash
git add src/services/audit/audit-skill.ts test/services/audit/audit-skill.test.ts
git commit -m "feat: orchestrate skill audits"
```

---

### Task 7: Format Human and JSON Audit Results

**Files:**
- Create: `src/adapters/in/cli/format-audit-result.ts`
- Test: `test/adapters/in/cli/format-audit-result.test.ts`

**Interfaces:**
- Consumes: validated `AuditResult` and `severityRank`.
- Produces: `formatHumanAuditResult(result: AuditResult): string` and `formatJsonAuditResult(result: AuditResult): string`.

- [ ] **Step 1: Write failing formatter tests**

Assert exact output for:

```text
Security score: 100/100
No security risks found.
```

and for a deliberately unsorted result:

```text
Security score: 42/100
CRITICAL: critical issue
HIGH: high issue
MEDIUM: medium issue
LOW: low issue
```

Also assert the input risk array is not mutated and JSON formatting equals `JSON.stringify(result)` with no ANSI sequences or extra text.

- [ ] **Step 2: Run formatter tests to verify they fail**

Run: `npm test -- test/adapters/in/cli/format-audit-result.test.ts`

Expected: FAIL because the formatter module does not exist.

- [ ] **Step 3: Implement deterministic formatters**

Sort a copied risk array by descending `severityRank`, retain provider order for equal severities through stable sort, uppercase the level label, and join lines with `\n`. Do not use Chalk: plain output is deterministic and satisfies both TTY and redirected use.

- [ ] **Step 4: Verify formatters and commit**

Run:

```bash
npm test -- test/adapters/in/cli/format-audit-result.test.ts
npm run typecheck
```

Expected: all formatting tests pass.

Commit:

```bash
git add src/adapters/in/cli/format-audit-result.ts test/adapters/in/cli/format-audit-result.test.ts
git commit -m "feat: format audit results"
```

---

### Task 8: Wire the `audit` Command and Exit Behavior

**Files:**
- Create: `src/adapters/in/cli/cli.ts`
- Modify: `src/index.ts`
- Test: `test/adapters/in/cli/cli.test.ts`
- Modify: `test/index.test.ts`

**Interfaces:**
- Consumes: `AuditSkill`, the two formatters, `CodexCliAuditEngine`, and `ExecFileProcessRunner`.
- Produces: `runCli(argv, dependencies): Promise<number>` and the executable `main(argv): Promise<number>`.

- [ ] **Step 1: Write failing CLI tests**

Inject an `AuditSkillPort`, stdout writer, and stderr writer. Cover:

- `audit <path>` calls the port once and writes exact human output plus one newline to stdout;
- `audit <path> --json` writes exactly one compact JSON object plus one newline to stdout;
- security scores `0` and `100` both exit 0;
- an `AuditError` writes `error: <publicMessage>` to stderr, nothing to stdout, and exits 1;
- an unexpected error writes `error: Unexpected audit failure`, nothing to stdout, and exits 1;
- missing path, extra positional arguments, and unknown options exit nonzero and never call the service;
- `--json` never changes operational failures into JSON;
- help exits 0 without calling the service.

- [ ] **Step 2: Run CLI tests to verify they fail**

Run: `npm test -- test/adapters/in/cli/cli.test.ts`

Expected: FAIL because `runCli` does not exist.

- [ ] **Step 3: Implement the Commander input adapter**

Create `src/adapters/in/cli/cli.ts` with:

```ts
export interface AuditSkillPort {
  audit(path: string, options?: { signal?: AbortSignal }): Promise<AuditResult>;
}

export interface CliDependencies {
  readonly auditSkill: AuditSkillPort;
  readonly stdout: (text: string) => void;
  readonly stderr: (text: string) => void;
  readonly signal?: AbortSignal;
}

export async function runCli(
  argv: readonly string[],
  dependencies: CliDependencies,
): Promise<number> {
  const program = new Command()
    .name("koloz")
    .exitOverride()
    .configureOutput({
      writeOut: dependencies.stdout,
      writeErr: dependencies.stderr,
    });
  let exitCode = 0;

  program
    .command("audit")
    .argument("<SKILL_MD_PATH>")
    .option("--json")
    .action(async (path: string, options: { json?: boolean }) => {
      try {
        const result = await dependencies.auditSkill.audit(path, {
          ...(dependencies.signal === undefined
            ? {}
            : { signal: dependencies.signal }),
        });
        const output = options.json
          ? formatJsonAuditResult(result)
          : formatHumanAuditResult(result);
        dependencies.stdout(`${output}\n`);
      } catch (error) {
        exitCode = 1;
        const message = error instanceof AuditError
          ? error.publicMessage
          : "Unexpected audit failure";
        dependencies.stderr(`error: ${message}\n`);
      }
    });

  try {
    await program.parseAsync([...argv]);
  } catch (error) {
    if (error instanceof CommanderError && error.exitCode === 0) return 0;
    return error instanceof CommanderError ? error.exitCode : 1;
  }
  return exitCode;
}
```

Use `exitOverride()` so parsing never calls `process.exit` in tests. Route Commander's help to stdout and parse errors to stderr. Catch errors once at this entry boundary, print only the typed `publicMessage` for `AuditError`, and return numeric status without logging diagnostics.

- [ ] **Step 4: Wire production dependencies and interruption**

Replace the Task 1 seam in `src/index.ts` with a composition root that constructs `ExecFileProcessRunner`, `CodexCliAuditEngine`, and `AuditSkill`. Create an `AbortController`, abort it on `SIGINT`, remove the listener after completion, and call `runCli` with writers backed by `process.stdout.write` and `process.stderr.write`. Set `process.exitCode` only in the executable guard.

- [ ] **Step 5: Run CLI and executable tests**

Run:

```bash
npm test -- test/adapters/in/cli/cli.test.ts test/index.test.ts
npm run typecheck
npm run build
node dist/index.js --help
```

Expected: tests, typecheck, and build pass; help includes `audit <SKILL_MD_PATH>` and `--json` and exits 0.

- [ ] **Step 6: Commit command wiring**

```bash
git add src/adapters/in/cli/cli.ts src/index.ts test/adapters/in/cli/cli.test.ts test/index.test.ts
git commit -m "feat: add audit CLI command"
```

---

### Task 9: Document Usage and Run End-to-End Verification

**Files:**
- Modify: `README.md`
- Create: `test/fixtures/skills/safe/SKILL.md`
- Create: `test/integration/codex-cli-audit.integration.test.ts`
- Modify: `vitest.config.ts`
- Create: `vitest.integration.config.ts`

**Interfaces:**
- Consumes: built `koloz audit` command and an optionally installed/authenticated Codex CLI.
- Produces: user installation/usage documentation, a default-suite exclusion for the nondeterministic integration test, and a manually runnable live-provider check.

- [ ] **Step 1: Add an opt-in integration test**

Create `test/integration/codex-cli-audit.integration.test.ts`. Invoke `main` against `test/fixtures/skills/safe/SKILL.md` with `--json`, capture stdout/stderr, assert exit 0 and empty stderr, parse stdout as JSON, and pass it through `validateAuditResult`. This file is reachable only through the dedicated integration configuration, so the default suite never invokes Codex.

- [ ] **Step 2: Exclude integration tests from the default suite**

Update `vitest.config.ts` so `npm test` excludes `test/integration/**`:

```ts
export default defineConfig({
  test: {
    environment: "node",
    include: ["test/**/*.test.ts"],
    exclude: ["test/integration/**"],
  },
});
```

Create `vitest.integration.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["test/integration/**/*.test.ts"],
  },
});
```

Add the cross-platform script:

```json
"test:integration:codex": "vitest run --config vitest.integration.config.ts"
```

to `package.json`.

- [ ] **Step 3: Expand the README**

Document prerequisites (Node.js 22+, npm, and an installed/authenticated `codex` for real audits), `npm ci`, `npm run build`, both output modes, score direction, exit semantics, recursive scope, symlink handling, and the opt-in integration command. Include examples:

```bash
koloz audit ./my-skill/SKILL.md
koloz audit ./my-skill/SKILL.md --json
```

- [ ] **Step 4: Run the full deterministic verification**

Run:

```bash
npm ci
npm test
npm run typecheck
npm run build
npm pack --dry-run
node dist/index.js audit ./missing/SKILL.md --json
```

Expected: install, tests, typecheck, build, and package dry-run pass; the package contains `dist/` and `schema/audit-result.schema.json`; the invalid-path smoke test exits nonzero, writes nothing to stdout, and writes one concise error to stderr.

- [ ] **Step 5: Optionally run the live Codex integration**

Run only when `codex` is installed and authenticated:

```bash
npm run test:integration:codex
```

Expected: the test exits 0 and validates a schema-conforming JSON audit. A missing credential or executable is an environment limitation, not a deterministic-suite failure.

- [ ] **Step 6: Commit documentation and verification assets**

```bash
git add README.md package.json package-lock.json vitest.config.ts vitest.integration.config.ts test/fixtures/skills/safe/SKILL.md test/integration/codex-cli-audit.integration.test.ts
git commit -m "docs: document skill audit workflow"
```

## Final Acceptance Checklist

- [ ] `npm ci`, `npm test`, `npm run typecheck`, `npm run build`, and `npm pack --dry-run` pass from a clean checkout.
- [ ] No production import of Commander exists outside `src/adapters/in/cli/cli.ts`.
- [ ] No production use of `exec`, shell-enabled `spawn`, or command-string interpolation exists; Codex arguments remain a discrete array passed through `execFile`.
- [ ] Both source and compiled validators load the canonical packaged schema.
- [ ] External symlinks are represented only as path/target metadata and their target content is absent from the snapshot.
- [ ] Every successful engine result is validated before formatting, and no operational failure is emitted as JSON.
- [ ] Snapshot cleanup is tested after success, engine failure, validation failure, and cleanup failure.
- [ ] Default tests never require network access, an installed Codex CLI, or authentication.
- [ ] The implementation conforms to ADRs 01 and 02, and the toolchain choices are recorded before they are used.
