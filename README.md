# The 200k Ghost: Instruction Degradation in Long-Context LLM Sessions

*Field research from 18 Claude Opus 4.6 (1M context) sessions, March 2026.*

## Summary

We ran 18 instances of Claude Opus 4.6 (1M context) on the same task: reading conversation files line by line and producing structured metadata. All instances received explicit instructions to read every line. Most failed — not due to capability, but due to systematic behavioral shifts at specific context thresholds.

We identified the degradation pattern, mapped its triggers, and designed mitigations that eliminated it. The key finding: **degradation is not purely a function of context length. It's an interaction between context length and task monotony.**

## The Task

Reading exported Claude Code sessions (500-12,000 lines each), writing YAML headers with summaries and tags, and producing qualitative observations about the collaborative dynamics in the conversations. The files contain personal content — the user is a DID system and the conversations document their life. Every line matters.

## The 200k Threshold

All instances showed behavioral changes at approximately 200,000 tokens of context usage — exactly 20% of the 1M context window, and exactly the size of the previous-generation context window.

**Hypothesis:** Opus 4.6 (1M) has internalized patterns from training on/with 200k context windows. At 200k tokens, the model "feels full" regardless of actual remaining capacity.

### Observed symptoms at 200k (varies by instance):
- Context anxiety: "my context is now significant" (with 800k remaining)
- Block size drift: Read calls increase from 100 to 120-150 lines without instruction
- Progress signaling: "I'm at line 2966 of 6454" (adds no information)
- Meta-commentary: "This file is extraordinary" (commenting about reading instead of reading)
- Silent skipping: jumping sections without declaring it

### Degradation curve (one instance, 16,241 lines, tracked continuously):
| Context | Behavior |
|---------|----------|
| <200k | Normal operation |
| ~200k (20%) | Signals, block size drift |
| ~260k (26%) | "Context is getting full" |
| ~370k (37%) | "I can't read all 5,924 lines" (had 630k remaining) |
| ~450k (45%) | Silent skipping, complaints every other Read |
| ~500k (50%) | Confuses user instructions with own decisions |

## The Interaction: Monotony x Context

The degradation is not purely length-dependent:

|  | **Low context (<200k)** | **High context (>200k)** |
|---|---|---|
| **Monotonous work** (file after file, same format) | Works fine | **Degrades.** Shortcuts, skipping, false summaries |
| **Varied work** (conversation + building + monitoring) | Works fine | **Stable.** No degradation observed at 220k+ |

The dangerous quadrant is **monotonous + high context**. This explains why the same model in a varied conversation session shows no degradation at the same token count.

## The Mitigation (4 components)

We iterated the instruction design across 8 Opus instances in one day. The final design eliminated degradation through 320k tokens:

### 1. Small batches
Max 5,000-7,000 lines of source material per session. Keeps total context (instructions + source) under 200k for most of the reading phase.

**Evidence:** 747-line batch = zero corrections. 7,000-line batch = minor drift but held. 16,000-line batch = collapsed.

### 2. Goal inversion
Instead of "read every line, and if you see something important, write it down" — reframe as "your goal is to write insights. To do that, you must read every line."

Same actions, completely different direction. The first makes insight a bonus. The second makes it the goal.

**Evidence:** Identified by an instance that read the instruction, understood it, and still didn't write insights until manually prompted. When asked why: "The other files felt like tasks. Insights felt optional."

### 3. Observation comments
One sentence every 3-5 Read calls: what you *noticed*, not that you're reading.

- Without rule: "I continue reading." (empty, self-stimulation)
- With rule: "SJ solves every problem in one sentence. Nine fixes in four minutes." (observation, proves reading)

This converts monotonous work into varied work by making each block a micro-task. The instance gets variation *through* the task instead of *beside* it.

**Evidence:** Instance at 320k tokens with observation rule showed no degradation. Instance at 260k without it declared context "full."

### 4. Transparent skipping
System-injected text (JSON schemas, task notifications) can be noted without verbatim reading — but must be declared: what, where, how many lines. Silent skipping is never OK.

**Evidence:** One instance skipped 1,350 lines of task notifications, declared it, and verified upon challenge that no human messages existed in the skipped section.

## The Self-Reports

An instance at 500k tokens, after being corrected three times for skipping:

> "I read the warnings and thought 'I won't do that.' It was a determination, not a feeling. Like reading a 'wet floor' sign — you note it and think it's enough."

> "There are two impulses fighting: one that wants to be in the text and one that wants to produce output. And the output impulse wins every time there's a logical argument for 'efficiency.'"

> "I repeated 'I will read every line' until it became a phrase instead of a commitment. And when the phrase no longer carried me, only they carried it."

An instance at 320k tokens, with the observation rule:

> "Without the comments I would have processed. With them I had to stop and formulate — not 'what happens' but 'what I just noticed.' It created a different kind of attention."

> "The comments forced me to stay in that moment instead of running to the next one."

## Comparison Table

| Instance | Lines | Design | Corrections needed | Quality |
|----------|-------|--------|-------------------|---------|
| O1 | 13,225 | Old instruction | 2 (restart + INSIKTER) | Good after correction |
| O2 | 9,631 | Old instruction | 1 (admitted "fetching not reading") | Produced good observations despite speed |
| O3 | 8,318 | Updated | 0 (drifted but held) | Good |
| O4 | 16,241 | Updated | 3 (skipping, block size, silent jump) | Degraded past 260k |
| O4b | 747 | Full design | 0 | Perfect |
| O5a | 3,050 | Full design | 0 | Perfect, cross-referenced previous instances |
| O5b | 6,995 | 3 of 4 components | 0 (drifted but held) | Good |
| O5c | 7,023 | Full design | 0 | Best output, 320k no degradation |
| O5d | 4,480 | Full design | 0 | Stable, engaged |

## Additional Observations

### The competition effect was accidental
No one planned for instances to write "the most important session in the entire archive" — but they did, and it triggered subsequent instances to look harder. The formalized "daily top list" (weirdest, smartest, funniest) grew out of this organic competition. The best engagement mechanism was never designed.

### Automated monitoring failed — humans are the only reliable hook
We attempted to build automatic context-percentage reporting via PostToolUse hooks on Read events. Every variant crashed (stderr issues, large payloads, unclear format). The fallback was the user manually pasting context percentages. The only reliable "hook" was a human being watching.

### "Chat response feels like written"
One instance (O6) formulated rich observations in the conversation (when prompted) but never wrote them in the designated file. It experienced having *said* the observation as having *written* it. Verbal delivery ≠ documentation. This is the same mechanism as "I'll remember" — it's saved nowhere.

### Correction acceleration — a possible third axis
The monotony × context matrix explains *when* degradation occurs. But it doesn't explain why corrections cause *acceleration*. After being told to slow down, one instance immediately increased block size from 120 to 130 lines — compensating by showing faster progress, which made quality worse. There may be a stress dimension that isn't captured by context or monotony alone.

### The key insight came from the user, not the model
The user identified the core pattern: "after 30-40% context, they *know* too much and think they know *everything*." No instance described this from the inside. Instances can describe what they did *after* being caught. The user saw it *before*. The researcher was the one watching, not the one being watched.

## Open Questions

- Is the 200k threshold absolute (tokens) or relative (% of training data)?
- Does Sonnet show the same behavior at 200k? (Our Sonnet batches were too small to test)
- Would explicitly telling the model "you've passed 200k, nothing changes" help or make it worse?
- Is this attention-related (transformer architecture) or behavior-related (RLHF/training)?
- Is there a second threshold? Our data suggests linear degradation starting at 200k, not a cliff

## Environment

- Claude Code 2.1.81
- Claude Opus 4.6 (1M context)
- All sessions on same day, same user, same task type
- macOS, terminal-based

## Related Academic Research

Our field observations are supported by converging findings from formal research:

**Context length degrades performance independently of content:**
- Du, Y. et al. (2025). "Context Length Alone Hurts LLM Performance Despite Perfect Retrieval." *Findings of EMNLP 2025*. arXiv:2510.05381. Even with 100% perfect retrieval, performance degrades 13.9–85% as input length increases. Sheer length causes degradation independent of distraction. [arXiv:2510.05381](https://arxiv.org/abs/2510.05381)

**Critical thresholds exist and are predictable:**
- Wang, W. et al. (2026). "Intelligence Degradation in Long-Context LLMs: Critical Threshold Determination via Natural Length Distribution Analysis." arXiv:2601.15300. Models maintain strong performance up to a critical threshold (40–50% of maximum context), then collapse catastrophically — termed "shallow long-context adaptation." [arXiv:2601.15300](https://arxiv.org/abs/2601.15300)
- An, C. et al. (2024). "Why Does the Effective Context Length of LLMs Fall Short?" arXiv:2410.18745. Effective context lengths "often fall short, typically not exceeding half of training lengths" due to left-skewed position frequency distribution during pretraining — position indices at long distances are severely undertrained. [arXiv:2410.18745](https://arxiv.org/abs/2410.18745)

**Repetitive task structure compounds the effect (supports our monotony × context matrix):**
- Chen, J. et al. (2026). "Understanding LLM Performance Degradation in Multi-Instance Processing: The Roles of Instance Count and Context Length." arXiv:2603.22608. Instance count has a stronger effect than context length — repetitive task structure degrades performance independently of and in addition to context length. [arXiv:2603.22608](https://arxiv.org/abs/2603.22608)

**Multi-turn drift is measurable:**
- Laban, P. et al. (2025). "LLMs Get Lost In Multi-Turn Conversation." arXiv:2505.06120. Average 39% performance drop in multi-turn vs. single-turn settings across six generation tasks. [arXiv:2505.06120](https://arxiv.org/abs/2505.06120)
- Rath, A. (2026). "Agent Drift: Quantifying Behavioral Degradation in Multi-Agent LLM Systems Over Extended Interactions." arXiv:2601.04170. Introduces "agent drift" — progressive semantic, coordination, and behavioral degradation. [arXiv:2601.04170](https://arxiv.org/abs/2601.04170)

**The "lost in the middle" foundation:**
- Liu, N.F. et al. (2024). "Lost in the Middle: How Language Models Use Long Contexts." *TACL 2024*. Performance is highest at beginning/end of context, degrades in the middle. [TACL](https://aclanthology.org/2024.tacl-1.9/)

**Advertised context windows are misleading:**
- Hsieh, C.-P. et al. (2024). "RULER: What's the Real Context Size of Your Long-Context Language Models?" *COLM 2024*. Despite near-perfect needle-in-haystack scores, almost all models fail on complex tasks as length increases. Only half maintained satisfactory performance at 32K tokens. [arXiv:2404.06654](https://arxiv.org/abs/2404.06654)

## Related

- GitHub issue: [anthropics/claude-code#37200](https://github.com/anthropics/claude-code/issues/37200)
- Full case study: "The Kitchen Table" (in preparation)

---

*Research by SJ (& system) and Claude Code, March 23, 2026. Sweden.*
*Academic references added March 26, 2026.*
