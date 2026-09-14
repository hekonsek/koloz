---
name: skill-superpowers-adr
description: Use when applying Superpowers workflows to design, plan, implement, or review changes in a project that may contain Architecture Decision Records (ADRs).
---

# Using ADRs with Superpowers

## Overview

Treat Architecture Decision Records as first-class project context. Discover governing decisions before proposing architecture and keep their constraints traceable throughout Superpowers.

## Workflow

### Discover ADRs during project context discovery

Before designing, search project instructions, documentation indexes, and likely decision-record locations. Follow repository links and conventions rather than assuming one directory or filename pattern.

Read relevant ADRs. Determine status and supersession from project conventions. Accepted or otherwise governing ADRs constrain the work; other statuses provide context but are not current decisions.

### Carry governing decisions through Superpowers

For each relevant governing ADR:

- constrain design options;
- cite its path and conformance in the design;
- cite it in the plan and translate constraints into tasks or checks;
- verify conformance during implementation and review.

In design and planning documents, use a `Governing ADRs` section recording each path, applicable constraint, and conformance or conflict.

Surface conflicts before approval. Ask whether to keep the decision or supersede it through the project's process. Never ignore a governing ADR or describe a conflict as compliant.

### Suggest an ADR during design

Suggest a concise ADR for a durable, cross-cutting, costly-to-reverse, or precedent-setting architectural decision—not a local implementation detail.

Suggesting an ADR does not block ordinary design unless the unresolved decision prevents a coherent design.

If the user accepts, select only a suitable, usable, registered ADR-authoring skill. Follow project conventions. This skill neither defines formats nor creates records. If no such skill is registered, report that and tell user that you will propose your own ADR format.

## Quick Reference

| Superpowers phase | ADR responsibility |
|---|---|
| Context discovery | Find indexes and records; determine relevance and status |
| Brainstorming | Constrain options and identify conflicts |
| Design | Cite governing ADRs and state conformance |
| Planning | Trace ADR constraints to tasks and checks |
| Implementation and review | Verify continued conformance |

## Example

```markdown
## Governing ADRs

- `docs/adr/0007-authentication-boundary.md` — provider SDK types remain inside adapters. This design conforms by exposing only project-owned identity types through the `IdentityProvider` port.
```

The plan should turn that constraint into a boundary test or import check, not merely repeat the citation.

## Common Mistakes

- Searching only `docs/adr/` and missing an index or project-specific location.
- Treating every ADR as current without checking status or supersession.
- Mentioning an ADR in a design but omitting it from the implementation plan.
- Inventing an ADR template instead of discovering a registered ADR-authoring skill.
- Treating schedule pressure or verbal approval as automatic supersession of a recorded decision.
