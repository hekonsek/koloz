# Use TypeScript for the CLI Application

## Context

Koloz is a CLI application that performs automated security audits of AI skills. Its code will need to represent structured inputs, audit findings, configuration, and command results while keeping command-line behavior predictable.

The project needs a language that supports the JavaScript package ecosystem used by CLI applications, makes contracts between these parts explicit, and catches common integration errors before a command is executed. The decision should cover the application's source language without prematurely selecting a specific runtime, CLI framework, or build tool.

## Decision

We will implement the Koloz CLI application in TypeScript. Application source files will use TypeScript, and the project will enable strict type checking. Executable artifacts may be emitted as JavaScript when required by the selected runtime and distribution approach.

Runtime, CLI framework, module format, build tooling, and packaging choices remain separate decisions.

## Consequences

Positive consequences:

- Static types will make the shapes of skill metadata, audit findings, configuration, and command results explicit.
- Strict type checking will catch many invalid assumptions and refactoring mistakes before users run the CLI.
- The project can use the JavaScript and Node.js-compatible package ecosystem while providing stronger editor support and code navigation.
- Shared domain types can provide consistent contracts across command parsing, audit logic, and output formatting.

Negative consequences:

- The development and release workflows require TypeScript-aware checking, build, or execution tooling.
- Type definitions and compiler configuration add maintenance work and can complicate use of packages with incomplete or inaccurate typings.
- Static types do not validate untrusted files or command-line input at runtime, so explicit runtime validation is still required.
- Contributors must understand TypeScript's type system and the distinction between source types and runtime behavior.

## Alternatives Considered

**JavaScript**. JavaScript would remove the TypeScript compilation and type-definition overhead and use the same package ecosystem. It was not selected because the CLI's structured audit data and expected evolution benefit from compile-time contracts and safer refactoring.

**A native compiled language such as Go or Rust**. These languages could produce self-contained binaries and provide strong static typing. They were not selected because they would give up direct access to the JavaScript ecosystem commonly used for AI-skill and package tooling, while introducing a different toolchain and a higher initial implementation cost.
