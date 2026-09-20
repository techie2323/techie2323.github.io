---
layout: post
title: "LLM Plays Satisfactory: Devlog, Part 2"
date: 2026-09-19
---

Picking back up from [yesterday's devlog](/2026/09/18/llm-plays-satisfactory-devlog.html) — the same project, an LLM-driven agent that watches and plays a live game of Satisfactory. Today moved from proving the toolkit works to actually using it to fix real problems in a real factory.

## Diagnosing three real production problems

Rather than a purpose-built test, this session started by pointing the agent at genuine flagged issues in an existing base. Three machines stood out:

- A smelter with both its input **and** output completely full — meaning nothing downstream was hauling ingots away, so it had simply stopped.
- A smelter with a completely empty inventory on both ends — no belt of any kind actually reached it.
- A constructor quietly set to make the wrong item entirely, with nothing feeding it at all.

Each diagnosis came from live monitoring data, not a guess. A machine's input buffer reading genuinely empty (not just low) turned out to mean something specific: it can only mean *no belt exists at all*, because a machine simply won't accept an item into its input buffer if that item doesn't match its current recipe — a belt carrying the wrong material would look just as empty as no belt at all. That distinction came from the user's own hard-won play experience, including a real story of accidentally cross-feeding one production line into another and having to rebuild the whole belt run to fix the contamination.

## A capacity trap, avoided

The obvious fix for the disconnected smelter was to tap it into the nearest existing ore supply. The numbers said otherwise: that miner was already producing exactly enough ore to feed two other smelters at their combined maximum, with essentially zero headroom. Adding a third consumer to that same fixed supply wouldn't have given the new smelter a real feed — it would have meant all three fighting over the same 60 items/minute, quietly dragging the two *already working* smelters down with it. The real fix was a dedicated new miner on an untouched ore node instead of a quick tap-in.

## Solving "which way does this thing point"

Building a miner → smelter → constructor → storage chain from scratch, with everything needing to line up physically, ran into a real gap: there was no reliable way to know in advance which face of a building was the input and which was the output, beyond one worked example from the day before.

The fix came from reading the game's own internal object structure directly rather than guessing a convention: every one of these buildings carries real connector components, each with its own exact position and facing direction relative to the building's center. Reading that data directly meant computing exact target placements instead of eyeballing them — and confirmed the same offset pattern holds across every building type tested so far (Smelter, Constructor, Storage Container), just scaled to each one's size. That's now a genuinely reusable rule, not a one-off fact about a single Smelter.

Placement itself still isn't perfectly predictable — the same aim-and-fire sequence sometimes landed exactly on target and sometimes drifted 100–300cm off, without an obvious pattern. The reliable fix wasn't preventing the drift, it was verifying the real result immediately afterward and retrying when it didn't land right — a repeatable loop rather than a first-try guarantee.

## A door that stayed closed, for now

One natural question came out of all that: the game shows a clear visual cue when two connectors are perfectly aligned — was that backed by any queryable state, or purely visual? Three real candidates were checked directly against the game's own data. One was real, but measuring the wrong thing (which floor tile a piece is standing on, not what it's aligned to). One read back empty every time it was checked. And the last simply didn't exist yet — that data only gets attached to a building once it's actually placed, not while it's still a floating preview.

So that specific question stays open. The fallback — place something, then check its real, measured position afterward — remains the dependable method, and it's the one that got a full, functioning production chain built end-to-end today.
