# Contributing

Thanks for helping improve Agent Claws. This repository holds the `CLAW.md` format specification, a set of example claws, and this guide. Contributions fall into two buckets, specification changes and example claws.

## Proposing a specification change

The specification lives in [SPECIFICATION.md](SPECIFICATION.md). To propose a change:

1. Open an issue describing the problem the change solves and the behavior you want.
2. Open a pull request that edits `SPECIFICATION.md` directly.
3. Keep the specification vendor-neutral. The format must be implementable by any runner, so it carries no mechanisms specific to a single product. Declarative, format-level features that any runner can honor are welcome; runner-specific execution behavior belongs in that runner's own documentation.
4. New top-level frontmatter keys are a breaking change for v1 parsers, which must reject unknown keys. Call out version impact in the pull request.

## Adding an example claw

Examples live in [examples/](examples/). Each example is one directory holding a single `CLAW.md` file that validates against the specification.

1. Add the file at `examples/<name>/CLAW.md` where `<name>`, the directory name, exactly matches the frontmatter `name` field. This layout is a convention for organizing examples in this repository, not something the format requires.
2. The file must validate against [SPECIFICATION.md](SPECIFICATION.md). At minimum it needs `name` and `description` and at least one `# Task` heading.
3. Keep examples vendor-neutral in their structure. A task prompt may name specific tools as an illustration, which is normal for examples, but the frontmatter should not depend on a single runner.
4. Set `license` so the example is clearly reusable. The examples in this repository use `MIT`.

## Style

The specification and these docs avoid colons in headings, em-dashes, and trailing periods on headings. Spell words out in full rather than abbreviating. When in doubt, match the surrounding text.
