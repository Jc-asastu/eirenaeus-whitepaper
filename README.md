# Eirenaeus-Philalethes
### A compound reasoning system on a single consumer GPU: same quality as its base model, 1.5x faster

J.C. - Asastu · Buenos Aires · October 2026 · draft v0.1

**[Read the designed PDF (11 pages, figures)](WHITEPAPER.pdf)**

---

## Abstract

Eirenaeus-Philalethes (Eirenaeus) is a local compound AI system. It wraps an open 27B reasoning model with a small
decision layer that runs on the CPU, a supervisor over the model's thinking, and a verified memory. It does not change
the model's weights.

On 677 problems from four public benchmarks (HumanEval+, MBPP+, AIME 2025, and a held-out mix of code and math), measured
on the same machine at temperature 0 against the same base model served alone, Eirenaeus shows **no statistically
significant difference in quality** (McNemar exact, p ≥ 0.62 on every benchmark) while being **1.51x faster in total
wall-clock time** (1.27x to 1.93x depending on the task). On tool calling (BFCL v4, 400 cases) it ties the base model
in quality but gives no speed advantage, and I report that too.

Underneath the numbers is a pattern rather than a product: a fast model that decides, beside a capable model that
reasons, each improving the other (Section 1). Everything here was measured on one RTX GPU with 12 GB of VRAM. I report
what worked, what did not, and the mistakes in my own measurement that I caught along the way.

**Key findings**
1. **Same quality.** No benchmark shows a significant difference; on HumanEval+ Eirenaeus solves two more.
2. **Faster where it matters.** The gain grows with how long the core would think: up to 1.93x on code.
3. **No edge on tool calls.** Short calls need no thinking; the base model with thinking off is just as fast.
4. **Method over luck.** One noisy run once suggested a loss that did not exist; every claim here is paired, at temperature 0.
5. **Memory that finds the right thing.** On a frozen exam the exact memory reaches the model 14 times in 15, with no private leak.

## 1. A different way to build

The interesting part of this work is not one product. It is a way to arrange models: a small, fast model that decides,
a *System 1* in the sense of fast and slow thinking, placed beside a capable reasoning model, a *System 2*, so that each
makes the other better.

Most systems scale one model, or route between whole models. Here neither model is replaced and neither is retrained to
start: the reasoning model keeps its weights, and the fast model only ever *chooses* among options. What changes is the
conversation between them.

- **What the fast model gives the reasoning model.** It decides how much to think before any GPU work starts, stops
  thinking that loops, picks the few memories worth injecting, and can open the reply with one plain line while the
  reasoning runs. Each choice costs about half a second of CPU and no GPU tokens.
- **What the reasoning model gives the fast model.** When the fast model is unsure, the reasoning model answers the same
  question, and the answer is logged as a training label. Today the fast model is confident in 30% of routing decisions
  (1,354 of 4,444); the rest fall back to a safe rule. Every escalation is material for the next version.

| What System 1 decides today | Instead of | Cost |
|---|---|---|
| How much to think: none, brief, deep | always thinking at full budget | ~0.5 s, CPU |
| Whether thinking is going in circles | running into the token cap | 0 ms, a rule |
| Which three memories to inject | search calls made by the model | ~0.45 s once |
| The opening line while it thinks | a blank screen for a minute | ~2 s, GPU |

**Why it generalizes.** Nothing here depends on these two models. The reasoning model is untouched, so any capable open
model can take the slow seat, and the fast seat learns from its own logged escalations, per user or per application. The
results in Section 6 are this pattern applied to one pair of models on one consumer GPU.

## 2. Motivation

A strong open model on a consumer GPU spends most of its time thinking. Much of that thinking is either unnecessary (the
answer was clear early) or misdirected (a simple request treated as a hard one). A system that decides *how much* to
think, and notices *when* it can stop, should save time without losing answers. The question is narrow on purpose:
**can an orchestration layer around a fixed model make it faster without making it worse?** The same weights, used
better.

## 3. Architecture

The parts are named after the three alchemical principles and the fifth essence. The names are a mnemonic, not a claim.

| Part | Role | Where it runs |
|---|---|---|
| **Mercury**: the decision layer (Laya, 421M parameters) | Reads each request in ~0.5 s and *selects*: how much to think, whether it is code, carries images, or needs live data | CPU |
| **Sulphur**: the reasoning core (Ternary-Bonsai-2-27B, PQ2_0, served by llama.cpp) | Thinks and answers | GPU, ~10 GB VRAM, ~65 tok/s |
| **Salt**: the body | One OpenAI-compatible endpoint, one model id, a local chat page | CPU |
| **Quintessence**: memory | Reads [Engram](https://github.com/Gentleman-Programming/engram) (Gentleman Programming); [Dokimos](https://github.com/Jc-asastu/dokimos) judges what is still valid; three memories are injected, never searched by the model (Section 7) | CPU |

**Three paths.** The *reply path* covers code, math and conversation: the core thinks with the effort the decision layer
chose, and a supervisor watching the thinking stream stops loops (the same doubt repeated three times) by asking for the
answer with what it has. Requests whose answer is the reply itself never receive memory and are never offered a tool
they do not need. The *light path* serves clients that bring their own tools (a browser, a file system, an application):
only the effort pick and a thinking cap. The *instant path* answers greetings and short facts without thinking.

## 4. Design principles

1. **Selection over generation.** The decision layer never writes text; it chooses among options. Choosing is cheap
   (~0.4 s on a CPU) and checkable; every escalation to the core becomes a training label.
2. **Think as much as it pays.** Thinking is the expensive part. A greeting gets none, a hard problem gets the whole
   budget, and a loop is stopped as soon as it repeats itself.
3. **Measure before claiming.** Every claim in this paper comes from a results file, never from a console, never from a
   single lucky run, and always against a control.

## 5. Evaluation methodology

- **Same machine, same harness, same questions** for both arms: Eirenaeus versus the base model served alone with the
  authors' default settings (thinking at maximum effort).
- **Temperature 0.** A single run at temperature 1.0 produced a "loss" on HumanEval+ (146 vs 156, p ≈ 0.002) that
  disappeared at temperature 0 (151 vs 154, p = 0.51). One sample per arm with randomness is not evidence.
- **Paired comparison** with McNemar's exact test on discordant items. "Better" or "worse" only below p = 0.05.
- **Held-out data.** 105 items development never used; the memory exam keeps a frozen judge half, run once per milestone
  and never tuned on.
- **The control arm.** When Eirenaeus seems to win thanks to a setting, the base model is also run with that setting.
  If the base model matches, the credit goes to the setting, not to Eirenaeus (see tool calling).
- **The GPU is not shared** during measurement; each step logs VRAM and any other GPU process.

## 6. Results

### 6.1 Reasoning: same quality, 1.51x faster

Temperature 0, RTX 12 GB, Eirenaeus v2 versus Ternary-Bonsai-2-27B alone.

| Benchmark | Items | Eirenaeus | Base alone | Only E / only base | McNemar p | Time E vs base | Speed |
|---|---|---|---|---|---|---|---|
| Held-out mix (code, GSM8K, AIME, MATH) | 105 | 97 | 98 | 1 / 2 | 1.00 | 6,604 s vs 8,561 s | 1.30x |
| MBPP+ | 378 | 308 | 309 | 7 / 8 | 1.00 | 12,227 s vs 19,228 s | 1.57x |
| AIME 2025 | 30 | 27 | 27 | 1 / 1 | 1.00 | 6,827 s vs 8,691 s | 1.27x |
| HumanEval+ | 164 | 156 | 154 | 3 / 1 | 0.62 | 5,055 s vs 9,752 s | 1.93x |
| **Total** | **677** | | | | | **30,712 s vs 46,231 s** | **1.51x** |

The gain lives in long problems. On short ones Eirenaeus carries a fixed cost of about one to two seconds per request
(median 12.3 s vs 11.2 s on the held-out mix), and wins back far more on problems where the core would otherwise think
for minutes.

### 6.2 Tool calling: a tie, and no speed advantage

BFCL v4, AST categories (simple, multiple, parallel, parallel-multiple), 100 each, official checker, temperature 0.

| Configuration | Correct / 400 | Total time | Median |
|---|---|---|---|
| Base alone, thinking (as published) | 373 | 1,589 s | 3.1 s |
| Base alone, thinking off | 371 | 877 s | 1.8 s |
| Eirenaeus, all layers | 362 (worse, p = 0.04) | 3,649 s | 5.1 s |
| Eirenaeus, light path, Mercury picks effort | **375** (tie, p = 0.75) | 2,027 s | 3.8 s |
| Eirenaeus, light path, thinking off | 368 | 1,072 s | 1.8 s |

Tool calls do not need thinking: the base model with thinking off is nearly as accurate and twice as fast. That speed
belongs to the setting, not to Eirenaeus, which adds about 0.2 s per call. With all its layers on, Eirenaeus was worse:
the date line in its prompt turned "this season" into the wrong year, a live-data note leaked into unrelated calls.
The light path fixes quality; it does not create an advantage.

**Using a computer, through the client.** Eirenaeus does not operate the computer itself. A client that brings its own
tools, such as a browser automation, a file system or an application connector, keeps them: Eirenaeus decides when and
how to call them on the light path, and the client executes them with its own safeguards. The tie above is the quality
of those calls.

### 6.3 Vision, and the current version

Images were the one place where Eirenaeus lost clearly. The cause sat in the decision layer, not in the core: it read
only the words of a request, so an exam question with a figure looked like a short factual answer and got no thinking.

| Benchmark | Items | Eirenaeus | Base alone | Only E / only base | McNemar p | Time E vs base | Speed |
|---|---|---|---|---|---|---|---|
| MMMU-Pro set 1 · v2 | 200 | 129 | 145 | 7 / 23 | < 0.01 | 11,021 s vs 9,318 s | 0.85x |
| MMMU-Pro set 2 · v3 | 200 | 130 | 137 | 9 / 16 | 0.23 | 7,256 s vs 7,848 s | 1.08x |
| HumanEval+ · v3 | 164 | 156 | 154 | 3 / 1 | 0.62 | 5,014 s vs 9,752 s | 1.94x |
| Held-out mix · v3 | 105 | 100 | 98 | 2 / 0 | 0.50 | 6,311 s vs 8,561 s | 1.36x |

Set 2 is 200 new questions never used in development; its times count only the 188 items where no other job used the
GPU. In v3 the decision layer receives a profile of the request (how many images, whether tools are offered), a request
with images never goes to "no thinking", and an exam answer with images travels without Eirenaeus's own context. The
significant loss of set 1 becomes no significant difference, and no advantage either.

A fix that broke the speed: the first v3 removed Eirenaeus's context from every direct answer, code included, and
HumanEval+ fell to 0.66x the base model's speed (one problem: 691 s instead of 22). That context is what keeps the core's
thinking short. Limited to images, the speed came back. MBPP+ and AIME 2025 were not re-run for v3; Section 6.1 is v2.

## 7. Memory: fewer, fresher memories, chosen for free

Memory is written, judged and chosen by three different parts. **Engram** writes: each memory is structured (what, why,
where, what was learned) and keyed by topic, so an evolving topic is updated instead of duplicated. **Dokimos** judges:
every memory expires and is re-attested against its source, and an expired or replaced memory never reaches the model.
**Eirenaeus** chooses: a privacy rule first, then the core rewrites the request as search keywords in both languages
(about 0.45 s, once per conversation), then a word search on the CPU injects the top three memories, about 200 tokens.
The model never spends a search call or a thought on remembering.

| Frozen memory exam (author's work memories) | Before | Now |
|---|---|---|
| The exact memory that answers is on the card (judge half, 15 questions) | 9 / 15 | **14 / 15** |
| Small talk and private questions get no memory (incl. 8 fresh probes) | 13 / 13 | **13 / 13** |
| Expired memories on the card (without Dokimos: 2) | 0 | **0** |
| Memories eligible after privacy, project and Dokimos filters (of 4,508) | 1,499 | 1,491 |
| Cost of choosing the card | 0.12 s CPU | 0.45 s, once |

The memories in this exam were written by the author's coding assistant, not by Eirenaeus; a memory Eirenaeus writes
for itself is on the roadmap. Privacy is a rule over a user-defined list, never the model's judgement.

## 8. What did not work

These are as informative as the wins, and each one changed the design.

- **Offering a tool the request does not need.** Asked to write code, the model kept calling a tool to "test" it; each
  refusal cost a new round of thinking, and one problem took 47 minutes. Removing the offer removed the loop.
- **Fixing a problem that did not exist.** A noisy single run suggested a code-quality loss; sending all code to maximum
  thinking cost the code speedup. The temperature-0 rerun showed there had been no loss.
- **Rewriting the conversation.** A first "answer while thinking" put its opening line back into the conversation with an
  instruction to correct itself; asked for a square root, the model invented another calculation. The opening line is
  now a separate request, and the reasoning sees exactly the user's words.
- **Privacy left to the model.** The core cannot know what is private to one person: when it wrote the memory keywords, a
  private topic reached the card once. Privacy became a fixed rule, checked on 8 fresh probes: 8 of 8.

## 9. Limitations

- One machine, one GPU class, one base model. The speedup belongs to this setup until measured elsewhere.
- Development touched public benchmarks. The held-out mix and judge split guard against overfitting to them.
- Short requests carry overhead. The decision layer costs about half a second per request.
- A small memory exam. Thirty questions on one person's work memories; a public memory benchmark comes next.
- No advantage on images. On MMMU-Pro it is within noise of the base model (130 against 137).

## 10. Roadmap

- **Next: verified early stop for code.** Stop thinking as soon as the code passes the examples in its own prompt.
- **Next: a memory it writes itself.** The decision layer flags what is worth keeping; the core writes it in the same
  structured form; Dokimos judges it.
- **Then: a public memory benchmark.** Long-term memory measured on conversations that are not the author's.
- **Then: distillation.** Train System 1 on System 2's logged answers, so the fast model decides alone more often.

## 11. Reproducibility

- Hardware: one NVIDIA RTX GPU with 12 GB VRAM; consumer desktop CPU; Windows 11.
- Base model: Ternary-Bonsai-2-27B PQ2_0 (llama.cpp, 64K context, reasoning budget 36,864, temperature 0 for evaluation).
- Decision layer: Laya, 421M parameters, CPU (revision 7b928d8 for the v3 runs, 55cf4c4 for the v2 table).
- Benchmarks: HumanEval+ and MBPP+ (EvalPlus v0.2.0), AIME 2025, BFCL v4 (official AST checker), MMMU-Pro, held-out mix
  with a fixed offset so development never saw it.
- Every number in this paper comes from a per-item results file (id, pass, seconds, route), available on request.

## Acknowledgements

Memory inspired by [Engram](https://github.com/Gentleman-Programming/engram) by Gentleman Programming; validity rules
after [Dokimos](https://github.com/Jc-asastu/dokimos).
