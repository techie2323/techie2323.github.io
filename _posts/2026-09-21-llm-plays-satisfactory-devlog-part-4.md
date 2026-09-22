---
layout: post
title: "LLM Plays Satisfactory: Devlog, Part 4"
date: 2026-09-21
---

Continuing from [Part 3](/2026/09/20/llm-plays-satisfactory-devlog-part-3.html) — an LLM-driven agent that watches and plays a live game of Satisfactory. Yesterday closed with one genuine loose end: the agent could see everything about Hard Drive research except make the actual choice of which reward to take. Today started by finally closing that gap, then moved into a brand new system altogether: power.

## Finding the missing piece by reading the source instead of guessing

The stuck point from yesterday was specific: the function that finalizes a Hard Drive claim never carried any information about which of the two offered rewards had actually been picked — only a generic placeholder. Every plausible function name on the obvious game object had already been tried and come up empty.

The fix came from changing approach entirely. Rather than continuing to guess live in-game, the actual (community-reverse-engineered) source code for the relevant game class was pulled directly from a public reference repository. Function bodies in that kind of generated reference file are empty stubs — but the function *signatures* are real, complete, and no longer a guessing game. That one file revealed the actual missing piece: each individual scanned Hard Drive is represented by its own object in the background, with its own "list my rewards" and "claim this one" actions built directly onto it — nothing generic about it at all. The dead end from yesterday turned out to be looking at the wrong object the entire time.

Built directly from that, the agent can now do the full loop end-to-end: list every drive currently sitting unclaimed, see exactly what each one is offering, and claim a specific reward from a specific drive — no manual clicking required. Verified for real: claiming a reward flipped the right flag in the save data, and a drive that had been claimed months earlier through the ordinary in-game menu — but never actually cleaned up by that path — was cleaned up correctly this way instead. That's a small but real bug in the base game's own bookkeeping, not something introduced by this project.

One more piece came out of that same investigation, explained directly by the user rather than reverse-engineered: rerolling. A drive that's been scanned but not yet claimed can be rerolled for an entirely new pair of options, instantly, with no wait — which is the whole reason experienced players sit on a stockpile of unclaimed drives rather than claiming each one as soon as it's ready. Confirmed live: a drive offering only one option (not enough unlocked recipes yet to draw a real pair) came back with two real options after a reroll, once more of the tech tree had been unlocked in between.

## A new system, from an empty grid

With research and progression fully closed out, the next target was power — specifically, wiring up a real coal power grid from a save that had none at all yet.

Power lines turned out to be a different kind of build action under the hood than everything built so far this project — not a simple "hold a preview, click once" placement, but a connect-two-points action, closer in spirit to how a pipe or belt gets run between two exact locations. Once that distinction was understood, though, it needed no new capability at all: the exact same two-step "aim at the first point, aim at the second point" approach already used for pipes worked immediately, once the real underlying recipe name was known. The proof it actually worked wasn't cosmetic — three previously disconnected generators all showed up afterward as genuinely sharing the same electrical circuit in the game's own live data, not just visually wired together.

Reading a building's real power connection point turned out to need the same kind of investigation done earlier in the project for belts and pipes — the existing tooling for finding "where exactly does this port sit in the world" only understood cargo and fluid connections, not power. Extending it to understand power connectors, plural connectors on power poles (which support several simultaneous wires — the higher tiers support quite a few at once), and a generator's separate coal and water connections closed that gap.

## A lesson that came from an honest mistake

Partway through, one placement attempt did nothing at all, silently. Once again, this was solved not by guessing but by trusting the person who actually knows how the game works: the fix was locking in an exact target position without also pointing the character's camera anywhere near it — the placement logic cares about where the camera is actually aimed, independent of any exact coordinate that's been locked in. Real, reproducible, and now written down so it doesn't get re-discovered the hard way a second time.

## Conveyor lifts, taught directly rather than reverse-engineered

The last stretch of the day was different in flavor from most of this project — instead of extensive guess-and-check, most of it was direct, hands-on teaching. The real placement flow for a lift turned out to be three actions, not two: anchor to a starting point, freely adjust and re-aim as many times as needed, then a final action that commits it — a small but real difference from the two-step flow used everywhere else so far.

A neat, real trick was demonstrated live: routing a lift straight through a special floor-hole fitting lets a belt line drop to a much lower elevation than a single lift segment could normally reach on its own, without needing to stack several lifts end to end. The fitting itself turned out to have two independent connection sides — top and bottom — each usable completely independently, which the agent learned to target correctly by watching a real, working example built specifically to demonstrate it.

The most interesting technical result of the day came from a direct question: can the agent actually see which way a lift is configured to move material, the same way a player can glance at the in-game arrow? It turns out yes — a real, readable value was found that matches exactly what's shown on screen. Trying to *change* that value directly, though, revealed something more interesting than simply "it worked" or "it didn't": the moment a lift is anchored to a real connection point, its direction snaps right back to whatever that connection demands, every single time, overriding any attempt to force it otherwise. That wasn't a bug getting in the way — it was the actual answer to the question. Direction isn't a free setting once a lift is connected to something real; it's *decided* by whatever it's plugged into, and only genuinely free-standing, unconnected lifts can be pointed either way on demand. A failed experiment, read correctly, turned out to be the most informative result of the session.

That's where today stopped: Hard Drive research now fully closed, from starting a scan through picking a specific reward — no human clicking needed anywhere in that loop anymore; a real, working power grid built from nothing; and enough now understood about conveyor lifts, including a genuinely subtle mechanic around direction, to build with them going forward.
