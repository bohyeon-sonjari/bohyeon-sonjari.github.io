---
layout: default
title: We blamed the model for weeks
---

# We blamed the model for weeks. Our harness was silently breaking its own KV cache.

*Notes from building [Sonjari](https://github.com/sonjari/sonjari), a free, open-source local agent.*

We're building a sub-agent on a local model. It drives the user's real Chrome, reads pages, and hands back what it found as an Excel sheet. It runs fine, but it felt slower than the Claude and Codex we use every day. For weeks we kept looking for the cause in the wrong places.

-Setup: Ollama, a 27B model (IQ3_S quantization + MTP), 128K context, one RTX 3090, Windows. Two of us build it together with Claude Code.

## Four causes we suspected

**Cause 1: text that changes every run was at the very front of the prompt.** That made the first call of every run re-read the whole prompt. We moved that text to the back. It got better, but it was still slow.


**Cause 2: turning thinking on and off in the middle of a run.** Across 955 logged model calls, the 20 where thinking was switched waited a median of 47.3 seconds for the first token. The 852 where it wasn't switched waited 1.9 seconds. It was a real cause, but it didn't happen often.


**Cause 3: a logging feature we'd left on.** We turned it off, and one prefill went from 32 s to 3.7 s. That was a single run, and we restarted the server to turn it off, so we never found out whether it was the feature or the restart.


**Cause 4: narrowing the tool list in the middle of a run.** The tool list sits in front of everything else in the prompt, so changing it makes the model re-read everything after it. One call took 93 seconds. We fixed that too.

After fixing all four, we settled on this: "It's a 27B on a consumer GPU, so it's just slow." The limitations list in our README even says "slower than cloud AI."

## We split the time

A comment on an earlier post suggested we separate the time the model spends from the time spent waiting on tools. We measured three runs that night.

- Tool time (mostly opening pages) was 7–10% of a run, about 2 seconds per page.
- A call whose history was unchanged spent 1–2 seconds on prefill (11–14 seconds when a big page was attached).
- A call where something earlier in the history had changed took 25–44 seconds each.

In one run of a Reddit research task, 15 of 38 calls were the third kind, where the earlier history had changed. They took 552 of 1,274 seconds: 43%.

## Earlier history has no reason to change

A page we already read is just text in the log. It can't change. But it was changing, and what changed it was our own harness.

To stay under the context budget, the harness shrinks old tool results. But on every call it picked again, from scratch, which ones to shrink. So the shrink boundary slid forward by one message per call, and the cache broke from that point on, every time.

## How we fixed it

Once we shrink something, we remember the shrunken form and send exactly the same thing on later calls. We shrink again only when the history really overflows, and then we do it in one big step. The history only ever grows at the end.

## What we measured after

First we replayed the same logs offline, without a model. The number of calls where earlier history changed dropped from 15 to 5, and the text re-read dropped by 61%.

Then the real test. We ran the same two tasks (a Reddit research task and a task researching 11 GitHub video tools) on the same PC, 3 runs each before and after the fix. Both tasks combined:

- Time spent re-reading the beginning of the prompt, per run (average): 327 s → 196 s
- Whole run (average): 721 s → 541 s (−25%)
- Plan completion (2.5/3 on average) and filled Excel rows (8.7 → 8.8) were about the same.

## How much to trust these numbers

- It's 3 runs per side. This is a direction, not a benchmark.
- The drop in whole-run time is mixed with a difference in step count. On the Reddit task the average went from 43 steps to 35 after the fix, and with 3 runs we can't tell whether the fix caused that. Per call, independent of step count, prefill went from 7.9 s to 5.8 s (−27%). For the calls where the earlier history had changed, it went from 174.8 s to 47.1 s (−73%).
- We never found out exactly why Cause 3 mattered.

## Something you can try

If your local agent is slow, log the prefill time of every call, and check whether the beginning of this prompt is identical, character for character, to the previous call's. Ours wasn't. The real cause wasn't the model, it was our own code.

## What we're chasing next: step count

Reading 11 GitHub repos took 56 steps. Every step pays for prefill, thinking, and writing. Sites with an API (GitHub, Hugging Face) don't need a browser at all, and for most pages we only use two or three fields. We'll post the numbers when we have them.

Has your harness ever been silently breaking its cache? Where was it?
