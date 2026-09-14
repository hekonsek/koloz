# skill-superpowers-adr: Skill for using ADRs with Superpowers

An [Agent Skills](http://skills.md) compatible, tool-agnostic skill that makes Architecture Decision Records first-class inputs to Superpowers workflows.

## How does it work?

The skill instructs agents to:

- discover existing project ADRs before design;
- respect governing decisions and surface conflicts;
- cite relevant ADRs in design and implementation plans;
- trace ADR constraints into implementation and verification;
- suggest an ADR for durable architectural decisions during design;

ADR structure, formatting, naming, creation, and validation are intentionally outside this skill's scope. We recommend using [ADR skill](https://github.com/hekonsek/skill-adr) for creating and maintaining ADR files themselves.