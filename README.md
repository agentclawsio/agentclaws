# Agent Claws

A standardized way to schedule AI agents to run recurring, unattended tasks.

## What are Agent Claws

A claw is a lightweight, open format for packaging work you would normally hand to a coding agent so it can run on its own. A claw is a single self-contained `CLAW.md` file. The file holds YAML frontmatter (`name` and `description` at minimum) followed by one or more ordered tasks. There are no companion directories, no sibling scripts, and no external assets. Everything a claw needs lives inside the one file.

```markdown
---
name: eng-dependency-cve-watch
description: Watch the security advisory feeds each morning for new entries that affect your declared dependencies, rank each by severity with the fixed version, and email a digest only when a new advisory lands.
schedule: daily @ 07:00
---

# watch

Check the GHSA, OSV, and NVD advisory feeds for new entries that affect the
dependencies you declare. Rank each by severity, note the affected range and the
fixed version, and email a short digest. If no new advisory touches your
dependencies, send nothing.
```

## Why Agent Claws

- **Unattended automation.** Anything you can ask a coding agent to do once, a claw can do over and over without you in the loop.
- **One portable file.** A claw is a single `CLAW.md` with no companion assets, so it copies, reviews, and version-controls like any other text file.
- **Scheduled or on demand.** Add a `schedule` and a runner executes the claw on its own cadence. Leave it out and the claw runs only when you ask.
- **Cross-product reuse.** The format is vendor-neutral, so the same `CLAW.md` runs on any compatible runner.

## How do Agent Claws work

**Progressive disclosure.** The frontmatter is cheap to scan, so a runner can load just the `name` and `description` of many claws for browsing and search. The body, which holds the full task prompts, loads only when a claw is activated for execution or detailed inspection. This keeps discovery fast even across a large collection of claws.

**Ordered execution.** A runner reads the tasks in source order and executes each one as an agent with full tool use. When a claw declares a `schedule`, the runner triggers it on that cadence. Without a schedule, the claw runs only on demand.

## Where can I use Agent Claws

Any CLAW.md-compatible runner can execute a claw. Because the format carries no vendor-specific mechanisms, a claw authored for one runner runs unchanged on another.

## Getting started

- **Specification.** Read [SPECIFICATION.md](SPECIFICATION.md) for the complete `CLAW.md` format.
- **Example Claws.** Browse [examples/](examples/) for working claws covering a range of schedules and runtimes.
- **Contributing.** See [CONTRIBUTING.md](CONTRIBUTING.md) to propose changes to the specification or add an example.

## Open development

Agent Claws was originally developed by Clor and released as an open standard so any runner can adopt it. The specification is vendor-neutral and the project welcomes ecosystem contributions. See [CONTRIBUTING.md](CONTRIBUTING.md) to get involved.

## License

Code in this repository is licensed under Apache-2.0 (see [LICENSE](LICENSE)). Documentation, including this README and the specification, is licensed under CC-BY-4.0.
