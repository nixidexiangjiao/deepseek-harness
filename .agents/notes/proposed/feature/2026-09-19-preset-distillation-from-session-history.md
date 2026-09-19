# Agent Note: Distilling agent presets from session history

Status: proposed

English | [中文](2026-09-19-preset-distillation-from-session-history.zh.md)

## Problem

Every preset is written by hand, while the evidence of what a preset should contain is already recorded and unread.

### How a preset works today

A **preset** is the configuration one agent session runs under. It is a directory whose `agent.cordis.yml` — its **composition** — names the plugins to mount: tools, prompt sections, and skills. [`dsh-agent-presets`](../../../../packages/preset/agent-presets/README.md) discovers presets from three roots, mounts one standing composition per preset, and lists a preset whose composition cannot load with the reason instead of hiding it.

```text
one session ──runs under──► preset ──────► composition (agent.cordis.yml)
                            (a directory)        │
                                                 └── mounts plugins:
                                                     tools, prompt sections, skills
```

Meanwhile every session writes a log. [`dsh-session-query`](../../../../packages/session-query/session-query/README.md) can already list, filter, read, and search those logs, so the record of which tools were called, which shell commands recurred, which files were read repeatedly, and where the user corrected the agent is available to application code today.

Nothing reads that record to answer the question the preset answers: what should this agent be configured as.

### Why the obvious fix is blocked

The gap is not a missing store. It is that the only way to author a preset is to copy an existing one whole.

[The authoring module](../../../../packages/preset/agent-presets/src/authoring.ts) accepts preset ids and an optional display name, never composition text, and it says why: authoring must grant no capability the copied preset did not already carry. A caller that could supply composition text could name any plugin, so accepting text would turn preset authoring into a way to mount anything.

Personalization therefore ends in hand-editing `agent.cordis.yml` after the copy. No record survives of which observation justified which line, and a second person has nothing to audit.

## Proposal

A distiller that reads session history, reduces it to counts, proposes a patch against a **base preset** the human names, and writes only after the human accepts — under one rule that keeps the authoring guarantee intact.

### The five stages

The five stages are scan, profile, propose, confirm, and write; only the last one touches disk.

```text
 ① scan              ② profile           ③ propose          ④ confirm         ⑤ write
 ─────────           ─────────           ─────────          ─────────         ───────
 session history     counts, not         a patch, plus      a human reads     copy(base, id)
 read through   ──►  transcripts:   ──►  the observations ──►  it and      ──► then apply
 dsh-session-query   tool histograms,    behind every       accepts or        the patch
                     recurring commands, proposed line      declines
                     hot files,               ▲                                    │
                     repeated corrections     │                                    ▼
                                         base preset,                        an ordinary
                                         named by the human                  preset directory
```

Stage ⑤ produces nothing special: the result is discovered, mounted, listed, and deleted by the existing roster with no new lifecycle.

### The capability floor

This is the rule the rest of the design hangs on. A distilled preset may add instruction data and may narrow what already exists; it may not mount a plugin its base does not mount.

```text
          base preset's mounted plugin set
          (computed by compositionInventory())
                        │
                        ▼
   ┌──────────────────────────────────────────────────┐
   │ the patch MAY                                    │
   │   add prompt sections    → dsh-persona           │──► written
   │   add file-backed skills → dsh-skill-filesystem  │
   │   narrow the config of a tool the base mounts    │
   ├──────────────────────────────────────────────────┤
   │ the patch MAY NOT                                │──► refused at
   │   mount any plugin the base does not mount       │    validation;
   └──────────────────────────────────────────────────┘    nothing is written
```

Prompt sections and file-backed skills pass the floor because they are instruction text, not capability grants: [`dsh-skill-filesystem`](../../../../packages/skill/skill-filesystem/README.md) discovers skills as files under scanned roots, and [`dsh-persona`](../../../../packages/preset/persona/README.md) registers prompt sections. Neither mounts anything new.

The floor is what preserves the authoring guarantee. Today that guarantee holds because composition text never comes from a caller. Under this proposal it holds because generated text is checked against the base's mounted plugin set before any write, so the result still grants nothing the base did not already carry.

### Where each part lives

| Part | Home | Why there |
|---|---|---|
| Corpus scan, floor check, write | the plugin | The floor is the security-relevant half and must fail loud in code. |
| Which pattern becomes a skill, a narrowed tool, or nothing | a skill the package ships | Classification rules stay readable and editable as instructions instead of compiled into `src/`. |

### A worked example

A user runs the `standard` preset for three months on one repository.

The scan counts 412 `bash` calls, of which 180 begin with `pytest`; 96 reads of `conftest.py`; and 14 turns whose next user message corrects the agent for running the whole suite instead of one file.

The distiller proposes one prompt section stating that this repository's tests run per file by default, and one skill recording how to select a test file. It proposes no tool changes, because no count supports one. Every proposed line is listed with the counts behind it.

The human accepts. The write is `copy('standard', 'py-repo')` followed by that patch. Nothing in the result mounts a plugin `standard` does not.

### Remaining mechanisms

- **Aggregate before reading.** The profile is built from counts using `listSessions`, `filterEvents`, and `searchEvents`. `readSession` is the selective escape for the few trajectories a count cannot explain. No run puts a whole session log into a model request.
- **Evidence-bound proposal.** A change with no observation behind it is not proposed at all. An absent field inherits the base, which is always the safe answer.
- **The preset file stays an input.** The mounted subtree already overrides `write()` as a no-op, so nothing here makes a preset a persistence target.
- **Corpus scope is declared per run.** The harness's own store is the default. Other stores that [the hooks bridges](../../../../packages/hooks/README.md) can reach are opt-in per run and named in the artifact, so a reader can tell which history produced which line.

Training-data extraction, routing signals, remote install of someone else's preset, and automatic re-distillation on a schedule are out of scope.

## Alternatives considered

- **Let the distiller write composition text directly:** rejected because it deletes the reason preset authoring is narrow. A caller that supplies composition text can mount any plugin, so a distiller misled by its own corpus — transcripts are text an attacker can reach — would become a capability-escalation path. The floor holds the worst case at "text the base could already produce".
- **Adopt RSIH's twelve-component ownership model:** rejected. That partition earns its value against a flat `settings.json`, where field ownership must be imposed from outside. Our configuration is a plugin graph in which each plugin's `Config` already partitions ownership, and a second fixed taxonomy would compete with the package groups with no gate maintaining it.
- **Adopt RSIH's compiled settings:** rejected. Rewriting declared keys into one settings file at startup forces a single active profile per process. The roster mounts one standing composition per preset and parents agent scopes to it, so sessions on different presets already run concurrently with separate state; compiling settings would give that up.
- **Ask the user what they want configured:** rejected as the path that already exists. The reason to read history is that stated preference and observed need diverge, and the observed side is the one no current view reports.
- **Ship the whole distiller as a skill, with no plugin:** rejected because a skill can only advise the model to respect the floor, never enforce it.

## Acceptance criteria

- A distilled preset mounts no plugin absent from its base composition; a patch that would is refused at validation with the offending plugin named, and no file is written.
- Every proposed line is accompanied by the observations that justify it, and the confirmation artifact lists them; a field with no supporting observation is absent from the output rather than defaulted.
- A run over a corpus of at least one thousand sessions issues no model request containing a whole session log.
- The written preset passes the existing roster's discovery and mount path unchanged, and `remove()` deletes it like any locally authored preset.
- Declining at the confirmation step leaves no preset directory and no partial write.
- Unit tests cover the floor check (accepting a prompt, skill, or narrowing patch; refusing a plugin addition), the aggregate-first scan, and the copy-then-patch write. A keyless recorded-session snapshot covers one end-to-end distillation, because the artifact and the confirmation exchange are model-visible.

## Risks

- **Private residue is a blocking precondition, not a follow-up.** Generated text is distilled from real transcripts and will carry absolute paths, internal hostnames, and material that looks like credentials. RSIH ships this gap knowingly and documents it; we should not. A redaction pass over generated text, with the residue shown at the confirmation step, belongs in the first landing.
- **The floor depends on a computable base plugin set.** `compositionInventory()` supplies it, but a base preset may gate rows on the host — the shipped `minimal` preset disables rows by `process.platform` — so the mounted set is host-dependent. The floor is computed on the resolving host and re-checked at mount, and a base whose inventory cannot be resolved is refused rather than approximated.
- **No gate asserts that a preset can express what the host composition can.** Presets carry tools, prompt sections, and skills; whether every configurable field is reachable from a preset is unchecked, so the distiller can only propose within whatever happens to be expressible. This proposal does not close that gap and will make it visible; closing it is a separate `process` decision.
- **Evidence skews toward recent and heavy use.** A profile built over an unbounded window lets one intense week define the agent. The aggregation window and the minimum number of occurrences before a pattern counts are `Config` fields changeable from cordis.yml, not constants, per the rule against hardcoded tunables in plugins.
- **A distilled preset is still trusted configuration.** The roster already requires treating every authored preset as trusted, because it grants the capabilities of the plugins it selects. The floor narrows what distillation can add; it does not make the result unreviewed, and the confirmation step is that review.
