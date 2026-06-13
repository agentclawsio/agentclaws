# Specification

> The complete format specification for CLAW.md

A **claw** is a subagent that automates work and runs in the background or on a schedule. Claws are just agents. Anything you can ask your coding agent to do once, a claw can do over and over. They use the same MCP servers, CLIs, and tools your agent does, so claws inherit whatever capabilities the host environment already provides. A claw with a `schedule` runs on that cadence; a claw with no `schedule` runs on demand, which a runner may expose as a manual run or an external trigger such as a webhook. The optional `concurrency` field governs what happens when runs would overlap, and it applies to every run source alike.

A claw is made of ordered tasks, plus a schedule, runtime defaults, and an optional shared system prompt. A CLAW.md file is the portable description of one claw, a single self-contained file with no companion directories, no sibling scripts, no external assets. Everything a claw needs lives inside it.

The format is designed for portability. Any tool can implement a CLAW.md parser and runner without knowledge of the host that produced it.

## CLAW.md format

A CLAW.md file has three parts, in order:

1. YAML frontmatter between `---` fences, holding claw-level metadata
2. Optional intro markdown between the closing fence and the first task heading
3. Zero or more tasks, each opened by a single top-level `# Task name` heading. A file with no tasks is valid documentation but cannot be executed

### Frontmatter

| Field | Required | Constraints |
|---|---|---|
| `name` | Yes | Max 64 characters. Lowercase letters, numbers, and hyphens only. Must not start or end with a hyphen, and must not contain consecutive hyphens. |
| `description` | Yes | Max 1024 characters. Non-empty. Describes what the claw does and when to use it. |
| `version` | No | Integer. The specification version the document targets. Absent means `1`. |
| `system_prompt` | No | Inline body of a shared system prompt applied to every task. |
| `schedule` | No | Schedule expression. Absent means the claw runs only when explicitly executed. |
| `start` | No | Earliest moment the schedule may trigger. A bare date `YYYY-MM-DD` or a datetime (see the `start` and `end` section). Absent means unbounded on the start side. |
| `end` | No | Latest moment the schedule may trigger. A bare date `YYYY-MM-DD` or a datetime; a bare date includes the whole day. Absent means unbounded on the end side. Must be greater than or equal to `start` when both are present. |
| `timezone` | No | IANA Time Zone Database name (e.g. `America/New_York`, `UTC`) governing clock rules and date-only `start`/`end` bounds. Absent means `UTC`. Interval rules ignore it. |
| `concurrency` | No | One of `skip`, `allow`, `queue`, `replace`. Governs what happens when a run is requested while another run of the same claw is active. Absent means `skip`. |
| `runtime` | No | Default runtime for tasks that do not declare their own. A short identifier with the same character and length rules as `name`. `auto` and `bash` are the conventional values; a host may define others. Absent defaults to `auto`. |
| `options` | No | Map of string keys to string values, passed to the runtime adapter. Every key is specific to the runtime that consumes it; the adapter maps the keys it recognizes to its CLI/config and may ignore or reject the rest. `model` and `effort` are common examples. Per-task `options` shallow-merge over these. |
| `skills` | No | Array of Agent Skills names the runtime should load. Max 64 characters per name. Lowercase letters, numbers, and hyphens, with no leading or trailing hyphen. Advisory; whether and how a runtime loads a named skill is an implementation detail, and it ignores names it does not recognize. A task that declares its own skills replaces this set for that task. |
| `timeout` | No | Default per-task wall-clock cap. Duration string of one or more `<count><unit>` segments, units `s`, `m`, `h`, non-negative integer counts, no fractional values (e.g. `45s`, `30m`, `1h`, `1h30m`). Absent or `0` inherits the runtime's built-in default. Tasks override per task. |
| `compatibility` | No | Max 500 characters. Free-form prose describing environment requirements (intended product, system packages, network access, credentials). |
| `license` | No | Short string. SPDX identifier or a short license string. |
| `metadata` | No | Map of string keys to string values for implementation-specific extensions. |

**Minimal example**

```markdown CLAW.md
---
name: claw-name
description: A description of what this claw does and when to use it.
---

# Task name

The prompt the agent will execute for this task.
```

**Example with optional fields**

```markdown CLAW.md
---
name: eng-dependency-cve-watch
description: Watch the GHSA, OSV, and NVD advisory feeds each morning for new entries that affect your declared dependencies, rank each by severity with the fixed version, and email a digest only when a new advisory lands.
schedule: daily @ 07:00
timezone: America/New_York
concurrency: queue
runtime: auto
options:
  model: fast
  effort: high
skills:
  - web-search
  - email
compatibility: A dependency manifest the agent can read. Private lockfile access is optional; without it the public manifest is the source of declared dependencies.
license: MIT
metadata:
  author: example-org
  version: "1.0"
---
```

#### `name` field

The required `name` field:

* Must be 1-64 characters
* May only contain lowercase alphanumeric characters (`a-z`, `0-9`) and hyphens (`-`)
* Must not start or end with a hyphen
* Must not contain consecutive hyphens (`--`)
* Is a stable identifier for the claw, suitable for unambiguous reference across systems. It is not a filename or directory name and need not match any containing path

**Valid examples**

```yaml
name: eng-dependency-cve-watch
```

```yaml
name: sales-funding-alerts
```

**Invalid examples**

```yaml
name: Sales-Funding-Alerts    # uppercase not allowed
```

```yaml
name: -sales-funding-alerts   # cannot start with a hyphen
```

```yaml
name: sales--funding-alerts   # consecutive hyphens not allowed
```

#### `description` field

The required `description` field:

* Must be 1-1024 characters
* Should describe both what the claw does and when to use it
* Should include keywords that help agents identify relevant tasks

**Good example**

```yaml
description: Sweep public funding signals each weekday against your target account list, then email each match with the round size, the lead investor where disclosed, and a one-line read on what the company can now afford.
```

**Poor example**

```yaml
description: Watches for funding.
```

#### `version` field

The optional `version` field declares which specification version the document targets:

* Is an integer
* Absent means `1`
* A parser must accept the versions it implements and reject a document whose declared `version` it does not support, so later features cannot be silently misinterpreted

**Example**

```yaml
version: 1
```

#### `system_prompt` field

The optional `system_prompt` field:

* Holds an inline body of a shared system prompt that every task inherits
* Is YAML text; multi-line bodies use the block scalar syntax (`|`)
* Is an inline body, not a reference to a named or externally stored personality. The full text lives here so the CLAW.md stays self-contained

**Example**

```yaml
system_prompt: |
  You are a cautious security operations assistant. Prefer clear,
  actionable findings over noisy speculation. Never modify firewall
  rules, users, packages, services, or files unless explicitly
  instructed.
```

#### `schedule` field

The optional `schedule` field specifies when a claw runs without explicit invocation. Grammar:

* `every Nm` or `every Nh` (interval rules; timezone-independent, anchored at 00:00 UTC)
* `hourly` (interval shorthand for `every 1h`)
* `daily`, `weekly`, or `monthly` (period shorthands firing at 00:00; `weekly` on Sunday, `monthly` on the 1st)
* `daily @ HH:MM[,HH:MM...]`
* `weekly @ HH:MM[,HH:MM...]`
* `monthly @ HH:MM[,HH:MM...]`
* `weekdays @ HH:MM[,HH:MM...]`
* `weekends @ HH:MM[,HH:MM...]`
* `mon,wed,fri @ HH:MM[,HH:MM...]` (specific days of the week)
* `on YYYY-MM-DD[,YYYY-MM-DD...] @ HH:MM[,HH:MM...]`
* Combine multiple rules with `;`

Clock rules (`daily`, `weekly`, `monthly`, `weekdays`, `weekends`, weekday lists, `on DATE`, and any `@ HH:MM`) are evaluated in the claw's `timezone`, which defaults to `UTC`. A bare `daily`, `weekly`, or `monthly` fires at 00:00 in that timezone (`weekly` on Sunday, `monthly` on the 1st), and a trailing `@ HH:MM` overrides the time. Interval rules (`every Nm`, `every Nh`, `hourly`) are timezone-independent. Absent means the claw runs only when explicitly executed, which a runner may expose as a manual run or an external trigger such as a webhook. The `concurrency` field governs overlap across all of these run sources alike.

Keywords and weekday tokens are lowercase (`mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun`). `@` introduces times. Times are 24-hour `HH:MM` in the range `00:00` to `23:59`. Optional spaces are allowed around `@`, `,`, and `;`. Rules combined with `;` are unioned, and duplicate trigger instants collapse to a single run.

`N` in an interval rule must be a positive integer. Interval rules anchor at 00:00 UTC of each day and reset at the next midnight, so an interval that does not divide 24h (e.g. `every 5h`) realigns at midnight and leaves a shorter final gap before 00:00. For even spacing, prefer a 24h-divisor interval: `1m`..`30m`, `1h`, `2h`, `3h`, `4h`, `6h`, `8h`, or `12h`.

**Examples**

```yaml
schedule: every 5m
```

```yaml
schedule: weekdays @ 09:00,17:00
```

```yaml
schedule: weekdays @ 09:00; weekends @ 12:00
```

```yaml
schedule: on 2026-01-15 @ 12:00
```

```yaml
schedule: monthly @ 09:00
```

```yaml
schedule: hourly
```

#### `start` and `end` fields

The optional `start` and `end` fields bound the window during which a `schedule` is active. The two fields are independent:

* `start` is the earliest moment the schedule may trigger. Before it, the claw does not run automatically.
* `end` is the latest moment the schedule may trigger. After it, the claw does not run automatically.
* An absent bound leaves that side unbounded.
* When both are present, `end` must be greater than or equal to `start`.

Each bound accepts one of these forms:

* a bare date `YYYY-MM-DD`
* `YYYY-MM-DDTHH:MM`
* `YYYY-MM-DDTHH:MM:SS`
* a full RFC3339 timestamp

A bare date and an offset-less datetime are interpreted in the claw's `timezone`; an RFC3339 string keeps its own offset. A bare-date `start` anchors at 00:00 of that day. A bare-date `end` resolves to the last instant of that day, so the end day is included.

These fields only affect scheduled triggers. A claw with no `schedule` runs only on explicit invocation (a manual run or an external trigger such as a webhook) regardless of `start` and `end`.

**Examples**

```yaml
schedule: daily @ 09:00
start: 2026-06-01
end: 2026-08-31
```

```yaml
schedule: every 1h
start: 2026-01-01
```

#### `timezone` field

The optional `timezone` field is an IANA Time Zone Database name (e.g. `America/New_York`, `Europe/Paris`, `UTC`). It governs every clock rule in `schedule` (`daily`, `weekdays`, `weekends`, weekday lists, `on DATE`, and any `@ HH:MM`) and the interpretation of date-only `start` and `end` bounds. Absent means `UTC`. Interval rules (`every Nm`, `every Nh`) ignore this field.

**Example**

```yaml
timezone: America/New_York
```

#### `concurrency` field

The optional `concurrency` field declares what a runner does when a new run of a claw is requested while another run of the **same** claw is active. Run requests come from any source equally, whether a `schedule` firing, a manual run, or a runner-exposed webhook. Identity is the claw's `name` within the runner's scope, so runs of different claws never gate one another.

The field is a fixed enum of four values:

* `skip` (the default). At most one run is active at a time. A request that arrives while a run is active is dropped.
* `allow`. Runs may overlap. Each request starts its own run with no limit on how many run at once. This suits a burst of webhook events that should each get their own run.
* `queue`. Runs never overlap, but a request during an active run waits and starts after the active run finishes. How many requests may wait, and whether repeated requests collapse while one is already waiting, is left to the runner.
* `replace`. A new request cancels the active run, then starts a fresh one. Cancellation is best effort. A runner stops the active run as cleanly as it can before starting the replacement.

An absent `concurrency` defaults to `skip`. The value is a fixed enum, so a parser rejects any other value rather than deferring it to the runtime. `concurrency` is a claw-level property of the whole run and has no per-task override.

**Example**

```yaml
concurrency: allow
```

#### `runtime` field

The optional `runtime` field names the runtime for tasks that do not declare their own. It is a short identifier following the same character and length rules as `name`: 1-64 characters, lowercase letters, numbers, and hyphens, with no leading, trailing, or consecutive hyphens.

Two values are conventional. Most runtimes are expected to support them, though support is optional:

* `auto` runs the task body as a prompt against an AI agent with full tool use. The model is selected through `options`.
* `bash` runs the task body as a shell script (it must be a single fenced ` ```bash ` block).

A host may also define more specific runtime identifiers, for example `claude` or `codex`. A runtime accepts the values it understands and rejects the rest at run time, not at parse time. Absent, `runtime` defaults to `auto`. A task may override this field via its per-task fenced YAML block.

#### `options` field

The optional `options` field is a map of string keys to string values that tune the runtime. Every key is specific to the runtime that consumes it. A runtime maps the keys it recognizes to its own CLI flags or configuration and may ignore or reject the rest. The spec defines no keys and does not validate the values; an unrecognized key or value surfaces at run time as an error from the runtime, not at parse time.

`model` (an opaque model identifier) and `effort` (an opaque reasoning-effort value such as `low`, `medium`, or `high`) are keys an `auto` runtime commonly accepts. They are illustrative; a runtime may accept any keys it chooses.

A task may override individual keys via its per-task fenced YAML block: per-task `options` shallow-merge over the claw-level `options`, so a task can change one key while inheriting the rest.

**Example**

```yaml
options:
  model: fast
  effort: high
```

#### `skills` field

The optional `skills` field is an array of skill names declaring the [Agent Skills](https://agentskills.io) a runtime should load for the claw. Each entry is a skill `name` (the identifier from a skill's `SKILL.md`): 1-64 characters, lowercase letters, numbers, and hyphens, with no leading or trailing hyphen. There is no slash-namespacing.

The format carries only the names. Whether a runtime supports Agent Skills at all, how it resolves a name to a skill, and whether or how it loads that skill are implementation details left to the runtime and its agent. The specification defines no skill names, no registry, and no resolution or loading behavior. A runtime maps the names it recognizes to its own catalog and ignores the rest, so a name is a portable hint, not a guaranteed contract. A runtime may honor or ignore the field as a whole.

Declaring a skill is a request, not a grant. A runtime still applies its own permissions, sandboxing, and credential policy, and never installs or runs anything merely because a claw names a skill. Naming a skill cannot widen what the runtime would otherwise allow.

Because `skills` is advisory, a name alone cannot guarantee a skill is present. When a skill is required rather than merely preferred, state that requirement in the human-readable `compatibility` prose as well, since the format does not mandate runtime execution behavior.

Order is not significant; `skills` is a set, and duplicate entries collapse to one. At the claw level, an empty `skills: []` is equivalent to an absent field. At the task level it is not, because a per-task `skills` block replaces the inherited set, so `skills: []` clears it (see the per-task replacement below).

A task replaces this set rather than merging into it. A task with no `skills` block inherits the claw-level set, and a task that declares `skills` replaces the claw-level set entirely for that task. This differs from `options`, which shallow-merges. The replacement supports least-privilege scoping: declare a high-reach skill only on the one task that needs it, or drop it for a task with `skills: []`.

**Example**

```yaml
skills:
  - web-search
  - email
```

#### `compatibility` field

The optional `compatibility` field:

* Must be 1-500 characters if provided
* Should only be included if the claw has specific environment requirements
* Can describe intended product, required system packages, network access, credentials, or anything else a host needs in place before running

**Examples**

```yaml
compatibility: Designed for a claw runner on a Linux host.
```

```yaml
compatibility: Requires git, docker, jq, and outbound HTTPS.
```

```yaml
compatibility: Requires Python 3.14+ and uv.
```

Most claws do not need this field.

#### `license` field

The optional `license` field specifies the license applied to the claw. Keep it short, either an SPDX identifier or a short license string.

**Examples**

```yaml
license: MIT
```

```yaml
license: Apache-2.0
```

```yaml
license: Proprietary
```

#### `metadata` field

The optional `metadata` field:

* Is a map of string keys to string values
* Lets implementations store properties not defined by this specification
* Must be preserved on round-trip when a tool reads and rewrites a CLAW.md
* Should use uniquely-named keys (e.g. `org:source`, `org:author`) to avoid accidental conflicts across implementations

**Example**

```yaml
metadata:
  author: example-org
  version: "1.0"
  org:source: https://example.com/claws/eng-dependency-cve-watch
```

### Body content

The Markdown body after the frontmatter contains the claw's tasks. Tasks are detected by their top-level `# ` headings.

#### Intro

Markdown between the closing frontmatter fence and the first `# ` heading is treated as a human-readable introduction. Implementations may render it on a detail page. It is never part of any task prompt and carries no runtime semantics.

#### Task headings

A task starts at the first column-0 `# ` heading (one hash followed by a space) and ends at the next column-0 `# ` heading or end of file. The heading text is the task name. Tasks run in the order they appear.

```markdown
# Watch

You are a watchman for this host. Gather these signals from the local
machine and write a brief report.
```

Task headings must be preceded by a blank line or start of body. A column-0 `# ` starts a task only when it is not inside a fenced code block, so a parser must track fenced code blocks while scanning for headings. Inside a task body, authors may freely use `##`, `###`, and lower-level headings without triggering a new task. A column-0 `# ` heading inside a task body would be interpreted as the next task; wrap such content in a fenced code block or use a lower heading level.

#### Per-task overrides

Immediately under a task heading (allowing one blank line) a fenced ` ```yaml ` block may set per-task overrides:

````markdown
# Watch

```yaml
runtime: auto
options:
  model: capable
skills:
  - web-search
```

You are a watchman for this host...
````

Recognised per-task keys: `runtime`, `options`, `skills`, `timeout`. If the block is absent, the task inherits the claw-level defaults. `options` is a map that shallow-merges over the claw-level `options` (see the `options` field definition above); the values are opaque and an unrecognized value surfaces at run time, not at parse time. `skills` is an array that **replaces** the claw-level `skills` for that task rather than merging into it (see the `skills` field definition above); a task with no `skills` block inherits the claw-level set, and `skills: []` clears it. This replace behavior is deliberately different from the `options` shallow-merge. `timeout` is a duration string of one or more `<count><unit>` segments, units `s`, `m`, `h`, non-negative integer counts, no fractional values (e.g. `45s`, `30m`, `1h`, `1h30m`); absent or `0` inherits the claw default, which itself inherits the runtime's built-in default.

A leading ` ```yaml ` block is read as overrides only when it is the immediate first content under the heading (after an optional blank line) **and** it contains at least one recognised override key. When it is the overrides block, any unrecognized top-level key inside it is a validation error, so a misspelled key is reported rather than silently swallowed. A leading YAML block that contains none of the reserved keys is treated as ordinary prompt body, so a prompt that merely opens with a YAML example is not consumed as overrides. The residual cost of this leniency is that a block carrying only a typo and no valid key (for example only `timeut: 5m`) reads as prompt body. A fenced YAML block is used here, rather than a second `---`-fenced frontmatter, to avoid colliding with markdown's horizontal-rule syntax.

#### Task body by runtime

* For `runtime: auto`, everything after the optional yaml block is the **prompt** sent to the model. Markdown formatting is allowed; implementations render it as plain text. A runtime other than `bash` follows this same prompt-body convention unless it defines its own.
* For `runtime: bash`, the body must contain exactly one fenced code block whose info string is exactly `bash` (an opening fence of three backticks at column 0, the info string `bash`, and a matching closing fence). The fenced content is the script. Prose outside the fence is treated as documentation and ignored at runtime.

````markdown
# Cleanup

```yaml
runtime: bash
```

Wipe the snapshot file written by the previous task.

```bash
#!/bin/bash
set -o errexit
set -o nounset
set -o pipefail
rm -f /tmp/snapshot.json
```
````

Tasks run in source order. How an implementation handles a task that fails, including whether later tasks still run, is left to the implementation.

### Placeholders

CLAW.md v1 treats `{{anything}}` as ordinary text. Tools neither extract nor substitute `{{...}}`. A tool that does not reformat the body preserves the body text exactly, whether it contains literal values or template-style markers; a tool that reserializes the frontmatter (for example rewriting `metadata`) need not be byte-identical.

Authors who want parameterization can hand-edit the file before running it. A future revision of this specification may define substitution semantics; v1 leaves that surface undefined.

## Progressive disclosure

A CLAW.md should be structured so agents can load detail incrementally:

1. **Frontmatter** is cheap to scan and is loaded cheaply for listing and search
2. **Body** loads when the claw is activated for execution or detailed inspection

Keep CLAW.md under 500 lines. If a task prompt needs extensive reference material, inline it concisely or split the work into multiple tasks.

## Validation

A conforming parser must reject:

* Missing `name` or `description`
* `name` violating the character or length rules
* `description` exceeding 1024 characters
* `compatibility` exceeding 500 characters
* `runtime` violating the `name` character or length rules (claw-level or per-task)
* A `skills` value that is not an array of strings, or any `skills` entry violating the identifier character or length rules (claw-level or per-task)
* A `concurrency` value that is not one of `skip`, `allow`, `queue`, or `replace`
* `end` earlier than `start` when both are present
* Unknown top-level frontmatter keys
* A present `version` whose value the parser does not support (e.g. `version: 2` on a v1 parser)
* A `bash`-runtime task whose body has zero or more than one fenced `bash` block
* A per-task overrides block (a leading ` ```yaml ` block carrying at least one recognised override key, one of `runtime`, `options`, `skills`, `timeout`) that also carries an unrecognized top-level key

A conforming parser must accept:

* An absent `version`, or `version: 1`
* A CLAW.md with no tasks (treated as documentation; cannot be executed)
* A CLAW.md with no intro markdown
* Unknown keys inside `metadata` (preserved on round-trip)
* Any keys inside `options` at either claw or per-task scope; the map is opaque in meaning and its values are strings, since nested maps and arrays are not part of the format
* An absent `skills` at either claw or per-task scope, and an empty `skills: []` (at claw level equivalent to absent; at task level an empty replacement set that clears the inherited skills)
* Duplicate `skills` entries (collapsed to one, since the field is a set)
* An absent `concurrency` (equivalent to `skip`)
* A `runtime` value other than `auto` or `bash` that satisfies the `name` character rules (the runtime, not the parser, decides whether it can run it)
* A leading ` ```yaml ` block that contains none of the recognised per-task override keys (treated as prompt body, not an error)
* Multi-line YAML block scalars in any string field
* `{{...}}` tokens anywhere in any string field, treated as literal text

## Version

This document specifies CLAW.md v1. The optional top-level `version` field is the forward-compatibility signal: a document declares the specification version it targets, an absent `version` means `1`, and a parser rejects a declared `version` it does not support. Alongside this, conforming v1 parsers must reject documents that introduce unknown top-level keys so that later features cannot be silently misinterpreted. The `skills` and `concurrency` fields are new top-level keys folded into v1 before adoption; a v1 parser recognizes them and must still reject any other unknown top-level key.
