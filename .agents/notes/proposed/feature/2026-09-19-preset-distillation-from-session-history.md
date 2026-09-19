# Agent Note: Distilling agent presets from session history

Status: proposed

English | [中文](2026-09-19-preset-distillation-from-session-history.zh.md)

## Problem

In one sentence: an agent's configuration is written by hand, although the sessions it has already run record what that configuration should say — and this note proposes reading that record, proposing a change from it, and writing the change only after a human accepts it.

### Terms used in this note

| Term | What it means here |
|---|---|
| DSH | DeepSeek Harness, this repository. Every package is named `@deepseek-ai/dsh-<name>`. |
| session | One conversation between a user and an agent, from start to end. Each session writes a log. |
| session log | The durable record of one session: every model request, tool call, tool result, and user message. |
| transcript | The text inside a session log. It contains whatever the user and the tools produced, so it is not trusted input. |
| plugin | One unit of behavior that can be switched on. A tool, a prompt section, and a skill provider are each plugins. |
| mount | To switch a plugin on for a given agent, so its registrations take effect. |
| preset | The configuration one agent session runs under. On disk it is a directory. |
| composition | The file `agent.cordis.yml` inside a preset directory, listing which plugins that preset mounts. |
| roster | The list of presets that [`dsh-agent-presets`](../../../../packages/preset/agent-presets/README.md) discovers, mounts, and can delete. |
| authoring | Creating or deleting a preset through the roster, as opposed to editing files by hand. |
| prompt section | A block of text added to the agent's system prompt, registered by [`dsh-persona`](../../../../packages/preset/persona/README.md). |
| skill | A file of task-specific instructions the agent can load on demand, discovered by [`dsh-skill-filesystem`](../../../../packages/skill/skill-filesystem/README.md). |
| base preset | The existing preset a distillation starts from. A human names it; the distiller never picks it. |
| patch | The set of changes this proposal would apply on top of a base preset. |
| capability floor | The rule limiting what a patch may contain. Defined under *The capability floor* below. |
| RSIH | RSI-Harness, a separate project solving the same configuration problem for a different agent. Referenced only under *Alternatives considered* below. |

### Functions named in this note

| Call | Owner | What it does |
|---|---|---|
| `listSessions()`, `filterEvents()`, `searchEvents()` | [`dsh-session-query`](../../../../packages/session-query/session-query/README.md) | Read session history without loading whole logs. |
| `readSession()` | `dsh-session-query` | Read one session log in full. |
| `copy(from, id, name)` | `dsh-agent-presets` | Copy an existing preset directory to a new id. The only write that creates a preset today. |
| `remove(id)` | `dsh-agent-presets` | Delete a locally authored preset. |
| `compositionInventory()` | `dsh-agent-presets` | Report which plugins each preset's composition mounts. |

### How a preset works today

```text
   session  ────►  preset  ────►  agent.cordis.yml  ────►  plugin  plugin  plugin
      ①              ②                   ③                        ④
```

| Mark | Element | Note |
|---|---|---|
| ① | session | Runs under exactly one preset, named explicitly or taken from the configured default. |
| ② | preset | A directory. The roster finds it in one of three roots and mounts one standing composition per preset. |
| ③ | composition | Lists the plugins to mount. A preset whose composition cannot load is listed with the reason rather than hidden. |
| ④ | plugins | Tools, prompt sections, and skills. What the agent can do is exactly what is mounted here. |

Every session also writes a log, and `dsh-session-query` can already list, filter, read, and search those logs. The record of which tools were called, which shell commands recurred, which files were read repeatedly, and where the user corrected the agent is therefore available to application code today.

Nothing reads that record to answer the question a preset answers: what should this agent be configured as.

### Why the obvious fix is blocked

The gap is not a missing store. It is that the only way to author a preset is `copy()` — a whole-directory copy of an existing one.

[The authoring module](../../../../packages/preset/agent-presets/src/authoring.ts) accepts preset ids and an optional display name, never composition text, and states why: authoring must grant no capability the copied preset did not already carry. A caller that could supply composition text could name any plugin, so accepting text would turn preset authoring into a way to mount anything.

Personalization therefore ends in hand-editing `agent.cordis.yml` after the copy. No record survives of which observation justified which line, and a second person has nothing to audit.

## Proposal

A distiller that reads session history, reduces it to counts, proposes a patch against a base preset the human names, and writes only after the human accepts — under one rule, the capability floor, that keeps the authoring guarantee intact.

### The five stages

```text
   ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐
   │ ①  │───►│ ②  │───►│ ③  │───►│ ④  │───►│ ⑤  │
   └────┘    └────┘    └────┘    └────┘    └────┘
                                    │
                                    └───► ✗ ───► ∅
```

| Stage | Name | What happens | Reads or writes |
|---|---|---|---|
| ① | scan | Read session history through `dsh-session-query`. | reads logs |
| ② | profile | Reduce it to counts: tool-call histograms, recurring shell commands, hot files, repeated user corrections. | in memory |
| ③ | propose | Build a patch against the base preset, with the observations behind every proposed line attached. | in memory |
| ④ | confirm | A human reads the patch and its evidence, then accepts or declines. | nothing |
| ⑤ | write | `copy(base, id)`, then apply the patch. | writes disk |
| ✗ | decline | The human declines at ④. | nothing |
| ∅ | — | No preset directory and no partial file are left behind. | — |

Only stage ⑤ touches disk, and what it produces is an ordinary preset: discovered, mounted, listed, and deleted by the existing roster, with no new lifecycle.

### The capability floor

This is the rule the rest of the design hangs on: a distilled preset may add instruction text and may narrow what already exists, but may not mount a plugin its base does not mount.

| What the patch wants to do | Allowed | Through | Outcome |
|---|---|---|---|
| Add a prompt section | yes | `dsh-persona` prefix and suffix | written |
| Add a file-backed skill | yes | the preset's own `skills/` directory | written |
| Narrow a mounted tool's own limits | yes | that tool's `Config` | written |
| Change any other config field | **no** | — | refused at validation, naming the field |
| Mount a plugin the base does not mount | **no** | — | refused at validation, naming the plugin; nothing is written |

The base's mounted plugin set comes from `compositionInventory()`, so "does the base mount this" is a computed answer rather than a judgement.

Permitted config edits are an explicit field allowlist, not "any field of an already-mounted plugin". A config field can widen behavior as surely as a new row, so the floor names the three it permits: `dsh-persona`'s `prefix` and `suffix`, the `customSkillDirs` entry that registers the preset's own bundled `skills/` directory, and a narrowing change to a mounted tool's own limits.

Prompt sections and file-backed skills pass the floor because they are instruction text, not capability grants: a skill is a file the agent may read, and a prompt section is text added to the system prompt. Neither mounts anything new.

The floor is what preserves the authoring guarantee. Today the guarantee holds because composition text never comes from a caller. Under this proposal it holds because generated text is checked against the base's mounted plugin set before any write, so the result still grants nothing the base did not already carry.

### Where each part lives

| Part | Home | Why there |
|---|---|---|
| Corpus scan, floor check, write | the plugin | The floor is the security-relevant half and must fail loud in code. |
| Which recurring pattern becomes a skill, a narrowed tool, or nothing | a skill the package ships | Classification rules stay readable and editable as instructions instead of compiled into `src/`. |

### A worked example

A user runs the shipped `standard` preset for three months on one Python repository. That preset already mounts `dsh-persona` and `dsh-skill-filesystem`, which is what makes the patch below legal.

Stages ① and ② read 143 sessions and reduce them to counts. Stage ③ turns those counts into this confirmation artifact, which is the whole of what stage ④ shows the human:

```text
preset distillation · base: standard · new id: py-repo
corpus: own store, 143 sessions, 2026-06-19 … 2026-09-19

[1] persona.suffix                                    +1 sentence
      Tests in this repository are run per file by default. Run the
      whole suite only when asked.
    evidence  180 of 412 bash calls begin `pytest`
              14 user turns correct a whole-suite run

[2] skills/select-test-target/SKILL.md                 new file
    evidence  96 reads of conftest.py across 61 sessions
              11 sessions re-derive the same file-selection steps

[3] skill-filesystem.customSkillDirs                   +1 entry
      <preset>/skills/
    reason    required by [2]; allowlisted field, plugin already mounted

not proposed: tools, model, runtime, policies, appearance
              no observation reached the configured minimum of 8

capability floor  OK · 0 plugins added · 3 allowlisted fields touched
                  accept / decline ?
```

The human accepts, and stage ⑤ runs `copy('standard', 'py-repo')` then applies the patch, producing this directory:

```text
~/.dsh/.agent-presets/py-repo/
├── preset.yml
├── agent.cordis.yml
└── skills/
    └── select-test-target/
        └── SKILL.md
```

Two rows of the copied `agent.cordis.yml` change, and both belong to plugins `standard` already mounts:

```diff
 - id: persona
   name: '@deepseek-ai/dsh-persona'
   config:
-    suffix: Your working directory is {{cwd}}.
+    suffix: >-
+      Your working directory is {{cwd}}.
+      Tests in this repository are run per file by default. Run the whole
+      suite only when asked.
     prefix: >-
       You are a coding agent powered by the {{model}} model.

 - id: skill-filesystem
   name: '@deepseek-ai/dsh-skill-filesystem'
+  config:
+    customSkillDirs:
+      - !!js "process.getBuiltinModule('node:url').fileURLToPath(new URL('skills/', baseUrl))"
```

The one new file is an ordinary skill bundle:

```markdown
---
name: select-test-target
description: Use when running tests in this repository, to choose which test file to run instead of the whole suite.
---

# Selecting a test target

Run one file with `pytest <path>`; run the whole suite only when asked.

To find the file for a change, locate the nearest `conftest.py` and the test module importing the changed module.
```

Had the same corpus also produced a proposal to mount a plugin — seven sessions asking about plugin internals might suggest `@deepseek-ai/dsh-tool-cordis`, which the `cordis` preset mounts and `standard` does not — the floor refuses it and the run writes nothing:

```text
[4] + plugin row '@deepseek-ai/dsh-tool-cordis'
    evidence  7 sessions ask about plugin internals

capability floor  REFUSED
  '@deepseek-ai/dsh-tool-cordis' is not mounted by base preset 'standard'
  nothing was written
```

Distillation cannot resolve that request; a human moves to a base preset that already mounts the plugin, or declines.

### Remaining mechanisms

- **Aggregate before reading.** The profile is built from counts using `listSessions`, `filterEvents`, and `searchEvents`. `readSession` is the selective escape for the few trajectories a count cannot explain. No run puts a whole session log into a model request.
- **Evidence-bound proposal.** A change with no observation behind it is not proposed at all. An absent field inherits the base, which is always the safe answer.
- **The preset file stays an input.** The mounted subtree already overrides `write()` as a no-op, so nothing here makes a preset a persistence target.
- **Corpus scope is declared per run.** The harness's own store is the default. Other stores that [the hooks bridges](../../../../packages/hooks/README.md) can reach are opt-in per run and named in the artifact, so a reader can tell which history produced which line.

Training-data extraction, routing signals, remote install of someone else's preset, and automatic re-distillation on a schedule are out of scope.

## Alternatives considered

- **Let the distiller write composition text directly:** rejected because it deletes the reason preset authoring is narrow. A caller that supplies composition text can mount any plugin, so a distiller misled by its own corpus — transcripts are text an attacker can reach — would become a capability-escalation path. The floor holds the worst case at "text the base could already produce".
- **Adopt RSIH's twelve-component ownership model:** rejected. That partition earns its value against a flat `settings.json`, where field ownership must be imposed from outside. DSH configuration is a plugin graph in which each plugin's `Config` already partitions ownership, and a second fixed taxonomy would compete with the package groups with no gate maintaining it.
- **Adopt RSIH's compiled settings:** rejected. Rewriting declared keys into one settings file at startup forces a single active profile per process. The roster mounts one standing composition per preset and parents agent scopes to it, so sessions on different presets already run concurrently with separate state; compiling settings would give that up.
- **Ask the user what they want configured:** rejected as the path that already exists. The reason to read history is that stated preference and observed need diverge, and the observed side is the one no current view reports.
- **Ship the whole distiller as a skill, with no plugin:** rejected because a skill can only advise the model to respect the floor, never enforce it.

## Acceptance criteria

- A distilled preset mounts no plugin absent from its base composition; a patch that would is refused at validation with the offending plugin named, and no file is written.
- Every proposed line is accompanied by the observations that justify it, and the confirmation artifact lists them; a field with no supporting observation is absent from the output rather than defaulted.
- A run over a corpus of at least one thousand sessions issues no model request containing a whole session log.
- The written preset passes the existing roster's discovery and mount path unchanged, and `remove()` deletes it like any locally authored preset.
- Declining at stage ④ leaves no preset directory and no partial write.
- Unit tests cover the floor check (accepting a prompt, skill, or narrowing patch; refusing a plugin addition), the aggregate-first scan, and the copy-then-patch write. A keyless recorded-session snapshot covers one end-to-end distillation, because the artifact and the confirmation exchange are model-visible.

## Risks

- **Private residue is a blocking precondition, not a follow-up.** Generated text is distilled from real transcripts and will carry absolute paths, internal hostnames, and material that looks like credentials. RSIH ships this gap knowingly and documents it; DSH should not. A redaction pass over generated text, with the residue shown at stage ④, belongs in the first landing.
- **The floor depends on a computable base plugin set.** `compositionInventory()` supplies it, but a base preset may gate rows on the host — the shipped `minimal` preset disables rows by `process.platform` — so the mounted set is host-dependent. The floor is computed on the resolving host and re-checked at mount, and a base whose inventory cannot be resolved is refused rather than approximated.
- **No gate asserts that a preset can express what the host composition can.** Presets carry tools, prompt sections, and skills; whether every configurable field is reachable from a preset is unchecked, so the distiller can only propose within whatever happens to be expressible. This proposal does not close that gap and will make it visible; closing it is a separate `process` decision.
- **Evidence skews toward recent and heavy use.** A profile built over an unbounded window lets one intense week define the agent. The aggregation window and the minimum number of occurrences before a pattern counts are `Config` fields changeable from cordis.yml, not constants, per the rule against hardcoded tunables in plugins.
- **A distilled preset is still trusted configuration.** The roster already requires treating every authored preset as trusted, because it grants the capabilities of the plugins it selects. The floor narrows what distillation can add; it does not make the result unreviewed, and stage ④ is that review.
