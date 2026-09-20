---
layout: post
title: "LLM Plays Satisfactory: Devlog, Part 3"
date: 2026-09-20
---

Continuing from [Part 2](/2026/09/19/llm-plays-satisfactory-devlog-part-2.html) — an LLM-driven agent that watches and plays a live game of Satisfactory. Up to this point the agent could build, run, and monitor a factory. Today was about closing a much bigger gap: progression. What's the point of an agent that can build anything, if it can't see what's in the research tree, can't pick a HUB milestone, and can't even hand-craft a screw?

## Closing the loop on research and milestones

The MAM (the game's research building) and the HUB (its milestone-and-delivery building) had already been readable for a while — the agent could see what was unlocked, what was in progress, what a milestone still needed. What it couldn't do was act on any of it.

That changed today, methodically, one real button at a time: starting a research project, selecting a HUB milestone, paying resources into it, and finally pressing the actual "launch pod" button that completes it. Each of those turned out to be a genuinely separate action under the hood, not steps of one bigger function — paying off a milestone's cost in full does *not* automatically complete it; a separate, distinct press is still required, discovered by watching for it and coming up empty on every other suspect.

One recurring headache resurfaced here: resolving a game object by its name string is not reliable for these particular class types, even for things that are unquestionably loaded and currently visible on screen. The fix, once it became clear this wasn't going away, was to stop resolving by name entirely wherever a live reference could be read directly off the game state instead — asking "what's currently active?" rather than "does this specific name exist?" That turned out to be the more dependable question every time.

By the end of that thread, the agent could drive the entire progression loop — start a research project, pick a milestone, pay it off, launch it — verified at every step against the game's own live data, not just trusted because a function call returned successfully.

## Teaching it to hand-craft

The next request was more basic and, it turned out, much harder: in the early game, before any machine exists, everything gets crafted by hand — mine the ore, then manually turn it into plates and rods through the crafting menu. Could the agent do that too?

Finding the right game object to control took a long detour through wrong guesses — half a dozen plausible-sounding "crafting manager" names, all nonexistent. The breakthrough came from an outside source: a reference to an old community-made crafting mod pointed toward the real underlying object, a small reusable "workbench" component that's actually attached to the player at all times, not tied to any physical bench. Once that was identified, the real crafting sequence fell out directly from watching it operate during an actual, real craft.

Then came two real crashes.

The first came from a basic mismatch with how the real game actually paces this action: production doesn't happen in one instantaneous step, it happens in small increments over real time, driven every frame while a button is held down. The first version tried to shortcut that by simulating hundreds of those increments back-to-back in a single instant — and the game did not like being told to do a full minute of production in zero real time. The fix was to make the agent's version work the same incremental way the real game does: one small step per call, called repeatedly with real pauses in between, exactly like holding a button down.

The second crash was subtler and only showed up once the first was fixed: completing a real craft cycle crashed the game again, but only for one specific item. The cause turned out to be pre-existing, unrelated clutter — that item's inventory stack had already been pushed well past its normal maximum by earlier, unrelated testing, and handing the game *more* of an already-overflowing stack broke something internal. Once confirmed against a completely different item with a normal, healthy stack, the exact same mechanism worked cleanly, with real material consumed and real output gained, matching the in-game inventory exactly.

## Hard Drives, and a mechanic worth understanding first

The last thread of the day was Hard Drive research — the mechanism behind alternate recipes, the optimized variants that can dramatically cut down on a resource you'd otherwise need a lot of. One example that came up directly: a late-game alternate that removes the need to manufacture screws at all for certain builds, which matters a great deal once you've seen how many screws a real base actually consumes.

Unlike normal research, a Hard Drive doesn't unlock anything the instant you start it — it runs on a real-world timer, several real minutes long, that keeps counting down even with the game closed. Once it finishes, the game offers a choice between two possible rewards, and whichever one goes unclaimed gets returned to a shared pool for a future drive. That's deliberate: the actual strategy players use is scanning many drives, banking the finished ones, and being selective about which to claim and when, fishing for specific recipes.

The agent can now do almost the entire loop that mirrors that real workflow: start a scan, poll the real countdown accurately (confirmed against the real in-game timer down to the second), and list exactly which two rewards are on offer before a decision gets made. What it still can't do is make that choice itself. The action that finalizes a claim was found and confirmed working — but it never carries any information about *which* of the two options was actually picked, no matter how many different real claims it was tested against. A separate broadcast was eventually found that does announce the real, specific choice after the fact — useful for confirming what happened, but not something that can be called to make the choice happen in the first place. The actual selection mechanism is still out there somewhere, unidentified, after working through every plausible name on the relevant game object without finding it.

That's where today stopped: a full, working, verified progression loop for research and milestones; real hand-crafting, debugged through two genuine crashes down to their actual root causes; and Hard Drive research readable end-to-end, with the last real piece — picking a specific reward — parked as clearly the next thing to chase.
