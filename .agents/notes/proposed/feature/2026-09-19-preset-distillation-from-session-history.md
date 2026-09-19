# Agent Note: Distilling agent presets from session history

Status: proposed

English | [中文](2026-09-19-preset-distillation-from-session-history.zh.md)

## Problem

Every preset is written by hand. A deployment that has run hundreds of sessions already holds the evidence of what its work needs — which tools get called, which shell commands recur, which files are read repeatedly, which corrections the user keeps repeating — and none of that evidence reaches the preset those sessions run under. [`dsh-session-query`](../../../../packages/session-query/session-query/README.md) can already list, filter, read, and search that history; nothing reads it to answer "what should this agent be configured as".

The gap is not a missing store. It is that the only authoring write today is a whole-directory `copy()` of an existing preset, and [the authoring module](../../../../packages/preset/agent-presets/src/authoring.ts) refuses composition text from a caller on purpose: authoring must grant no capability the copied preset did not already carry. Personalization therefore ends in hand-editing `agent.cordis.yml`, with no record of which observation justified which line, and no way for a second person to audit the result.

## Proposal

A distiller that reads session history, aggregates it into a usage profile, proposes a preset derived from a human-named **base preset**, and writes only after the human confirms — under a capability floor that keeps the authoring guarantee intact.

- **Capability floor — the load-bearing rule.** A distilled preset may add prompt sections through [`dsh-persona`](../../../../packages/preset/persona/README.md), add file-backed skills that [`dsh-skill-filesystem`](../../../../packages/skill/skill-filesystem/README.md) discovers, and narrow the configuration of tools the base already mounts. It may not mount a plugin the base composition does not mount. The authoring invariant holds today because composition text never comes from a caller; under this proposal it holds because generated text is checked against the base's mounted plugin set before any write. Prompt sections and file-backed skills are instruction data, not capability grants, which is what lets them through the floor.
- **Aggregate before reading.** The profile is built from counts first — tool-call histograms, recurring shell-command prefixes, hot file paths, and repeated user corrections — using `listSessions`, `filterEvents`, and `searchEvents`. `readSession` is the selective escape for the few trajectories a count cannot explain. No run loads whole session logs into a model request.
- **Evidence-bound proposal.** Each proposed change carries the observations that justify it, and a change with no observation behind it is not proposed at all. An absent field inherits the base, which is always the safe answer.
- **Judgement in a skill, enforcement in the plugin.** The plugin owns the corpus scan, the floor check, and the write. Which recurring pattern should become a skill, which a narrowed tool, and which is mere preference belongs to a skill the package ships, so the classification rules are readable and editable as instructions rather than compiled into `src/`.
- **Write is copy-then-patch.** Confirmation runs `copy(base, id, name)` and then applies the patch confined to the floor. The preset file stays an input: the mounted subtree already overrides `write()` as a no-op, so nothing here makes a preset a persistence target.
- **Corpus scope is declared per run.** The harness's own store is the default. Other stores that [the hooks bridges](../../../../packages/hooks/README.md) can reach are opt-in for a run and named in the artifact, so a reader can tell which history produced which line.

A distilled preset is ordinary configuration once written: it is discovered, mounted, listed, and deleted by the existing roster with no new lifecycle.

Training-data extraction, routing signals, remote install of someone else's preset, and automatic re-distillation on a schedule are out of scope.

## Alternatives considered

- **Let the distiller write composition text directly:** rejected because it deletes the reason the authoring surface is narrow. A caller that supplies composition text can mount any plugin, so a distiller compromised by its own corpus — transcripts are attacker-reachable text — would be a capability-escalation path. The floor keeps the blast radius at "text the base could already produce".
- **Adopt RSIH's twelve-component ownership model:** rejected. That partition earns its value against Pi's flat `settings.json`, where field ownership must be imposed. Our configuration surface is a plugin graph in which each plugin's `Config` already partitions ownership, and a second fixed taxonomy would compete with the package groups without any gate maintaining it.
- **Adopt RSIH's compiled settings:** rejected. Rewriting declared keys into a single settings file at startup forces one active profile per process. The roster mounts one standing composition per preset and parents agent scopes to it, so sessions on different presets already run concurrently with separate state; compiling settings would give that up.
- **Ask the user what they want configured:** rejected as the path that already exists. The reason to read history is that stated preference and observed need diverge, and the observed side is the one no current surface reports.
- **Ship the whole distiller as a skill, with no plugin:** rejected because a skill cannot enforce the capability floor — it can only advise the model to respect it. The floor is the security-relevant half and belongs in code that fails loud.

## Acceptance criteria

- A distilled preset mounts no plugin absent from its base composition; a generated patch that would is refused at validation with the offending plugin named, and no file is written.
- Every line the distiller proposes is accompanied by the observations that justify it, and the confirmation artifact lists them; fields with no supporting observation are absent from the output rather than defaulted.
- A run scanning a corpus of at least one thousand sessions issues no model request containing a whole session log.
- The written preset passes the existing roster's discovery and mount path unchanged, and `remove()` deletes it like any locally authored preset.
- Declining at the confirmation step leaves no preset directory and no partial write.
- Unit tests cover the floor check (accepting a prompt/skill/narrowing patch, refusing a plugin addition), the aggregate-first scan, and the copy-then-patch write; a keyless recorded-session snapshot covers one end-to-end distillation, since the artifact and the confirmation exchange are model-visible.

## Risks

- **Private residue is a blocking precondition, not a follow-up.** Generated text is distilled from real transcripts and will carry absolute paths, internal hostnames, and material that looks like credentials. RSIH ships this gap knowingly and says so; we should not. A redaction pass over generated text, with the residue surfaced at the confirmation step, is part of the first landing rather than a later hardening.
- **The floor depends on a computable base plugin set.** `compositionInventory()` supplies it, but a base preset may gate rows on the host — the shipped `minimal` preset disables rows by `process.platform` — so the mounted set is host-dependent. The floor is therefore computed on the resolving host and re-checked at mount, and a base whose inventory cannot be resolved is refused rather than approximated.
- **No gate asserts that a preset can express what the host composition can.** Presets carry tools, prompt sections, and skills; whether every configurable surface is reachable from a preset is currently unchecked, so the distiller can only propose within whatever happens to be expressible. This proposal does not close that gap and will make it visible; closing it is a separate `process` decision.
- **Evidence skews toward recent and heavy use.** A profile built over an unbounded window lets one intense week define the agent. The aggregation window and the minimum support for a pattern are `Config` fields changeable from cordis.yml, not constants, per the no-hardcoded-tunables rule.
- **A distilled preset is still trusted configuration.** The roster already requires treating every authored preset as trusted because it grants the capabilities of the plugins it selects. The floor narrows what distillation can add; it does not make the result unreviewed, and the confirmation step is the review.
