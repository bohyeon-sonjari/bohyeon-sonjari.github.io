---
layout: default
title: "Got a 35B model running on 6GB"
---
Previous: [We blamed the model for weeks. Our harness was silently breaking its own KV cache.](/)

# I was about to give up on running a sub-agent on 6GB. Then I got a 35B model running on it.

We're building a sub-agent called Sonjari that runs on a local model. This is the story of trying it on a laptop with an RTX 3050 6GB and 24GB of RAM.

With a 4B model (3-bit), we ran the same task 19 times and barely got anywhere. On a price-finding task (3 tasks, 9 cells), it filled only 1.0 cells on average. So we were ready to give up on running an agent on this card.

Then we had an idea. A mixture-of-experts (MoE) model only uses a few of its experts at a time. So what if we keep the whole model in RAM and put only the parts we need right now on the GPU? We downloaded Qwen3.6-35B-A3B at 2-bit (UD-IQ2_M, an 11.5GB file) and tried it.

## We threw away the first run by mistake

I launched the engine from the wrong folder, so it never picked up the GPU and ran on CPU only. The first run was invalid. Once I fixed the working folder, it found the card again.

## Speed

We measured with a 29,350-token input at the start.

- First read: 92-95 s (default loading) or 72 s (`--load-mode none`)
- Re-reading the same beginning: 3.4-3.7 s
- Writing: 25-30 tokens per second

Converting the first read to tokens per second gives roughly 310-410 (a calculated number, not something we measured separately). When the cache matches, re-reading takes about 3 seconds, so keeping the beginning of the prompt unchanged mattered a lot on this laptop.

## The real problem was RAM

With `--load-mode none`, committed memory was at 99%. When Chrome opened while Sonjari was running, the engine crashed with `std::bad_alloc`. All 3 runs from that batch were invalid.

We suspect two causes. The engine keeps memory of the beginning of the prompt in RAM, up to 8GB by default (`--cache-ram 8192`), and it keeps 32 rewind points. These are suspicions. We did not turn them off one by one to check.

When we switched to `--cache-ram 0 --ctx-checkpoints 4` with default loading, it survived alongside Chrome. Free RAM stayed between 1.7 and 5.5GB, and it crashed 0 times.

## Accuracy

On the price-finding task (3 tasks, 9 cells), the 35B model filled 5, 2 and 1 cells in its 3 runs. That's an average of 2.7. It's better than the 4B's 1.0, but honestly it still gets a lot wrong. In one run, 5 cells were filled correctly, 3 were lost, and 1 was misread.

## One mistake of ours

When we first posted these results on Reddit, we wrote "35B: 5/9 in 1 run, 4B: 18 runs." After a commenter's question, we recounted from the original result files. For the 35B there were two more files from the same day (3 runs: 5, 2, 1), and the 4B had 19 runs, not 18. We corrected the numbers. We almost said "this model works" based on one run of 5/9.

## How far to trust these numbers

- The 35B has only 3 runs, and they vary a lot (5, 2, 1). The 2.7 average is a direction, not a result.
- The difference between the 4B and the 35B isn't only the model. The 4B runs are from October and the 35B runs are from a November beta build, so fixes to our harness in between are mixed in.
- One 35B run took 918 s (the first run), while the 4B took 351-466 s. So it was about 2x slower.
- We measured on one card in one laptop.

## What you can try

If you have a 6GB card and plenty of RAM, an MoE model split across RAM and GPU is worth trying. And if it crashes, look at RAM usage first, not GPU memory. For us, the default that keeps the beginning of the prompt in RAM was the suspect.

## What we'll look at next

We downloaded a smaller file (UD-IQ2_XXS, 10.7GB) but haven't used it yet. On a 16GB-RAM laptop this 35B may not load at all, so we'd need a smaller MoE.

If you run agent work on a 6GB-class GPU, which model do you use? Has anyone tried splitting an MoE across RAM and GPU?

(The agent we're building is Sonjari. It's free and open source: https://github.com/sonjari/sonjari)
