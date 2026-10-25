# Build Journal — `forged`

A local, private, retrieval-grounded security assistant. v2 of an earlier local-RAG project, built for authorized lab work, CTF practice, and study on a thin-and-light Intel laptop with no NVIDIA GPU. Working directory name: `forged`.

This journal documents the build start to finish. Chapters are written as work happens. Chapter 0 is the planning backfill — everything that happened before the first line of code.

---

## 2026-10-10 — v2 direction reset

The v2 rebuild drops the earlier "uncensored" framing. forged is now a local, private, retrieval-grounded security assistant for authorized lab work, CTF practice, and study. It behaves like a well-aligned assistant; the reasons it runs locally are privacy, offline operation, and domain specialization, not the removal of safety behavior.

What this changes going forward:

- Model selection favors the strongest well-aligned, security-capable local models rather than ruling a model out for declining harmful requests. Candidates such as Cisco Foundation-Sec-8B-Reasoning are back in scope on knowledge merits.
- Any fine-tuning targets domain knowledge and answer format, grounded in the public corpus, not the removal of refusals.
- The rest of the stack is unchanged: hybrid retrieval, the knowledge-graph layer, the sandboxed tool loop for your own authorized lab, and the eval harness that gates every change.

The chapters below are preserved as the historical record of the v1 and early-v2 build. Where they describe model choices made on the earlier framing, read them in light of this reset.

---


## Chapter 0 — The Setup

> *"i found out about a 14B model that might be usefull NousCoder-14B."* — and three sub-agents later, we were still recommending the same model we started with. The setup phase of this project was, on paper, "pick a model and start building." In practice, it was three weeks of cross-checking, rejecting tempting answers, and exposing a series of small traps that would have quietly degraded the final product if we'd taken any of them. This chapter is about those traps.

### Where we started

Project v1 was already a working local cybersecurity assistant — a Textual TUI on top of an Ollama-served local model, with a hybrid RAG stack (ChromaDB + BM25s + a cross-encoder reranker) over HackTricks, PayloadsAllTheThings, the OWASP Cheat Sheet Series, and MITRE ATT&CK. The model was `huihui_ai/deepseek-r1-abliterated:14b`. The embeddings were `nomic-embed-text`. The reranker was `cross-encoder/ms-marco-MiniLM-L-6-v2`. The TUI exposed THINK/RAG/CTX toggles and a TOPK control.

It worked. But by May 2026, every component above was 1–2 generations behind state of the art, the agent could only *describe* offensive operations rather than perform them, there was no knowledge graph layer to handle CVE/ATT&CK relationships, the reasoning was constrained by a model whose `<think>` toggle behavior had degraded under abliteration, and the TUI ceilinged out at chat — no tool-call cards, no citation panel, no findings graph, none of the surfaces a 2026 AI product is expected to have.

The choice was either to ship a thin "model swap" patch or do a real v2. We chose v2.

### The hardware reality

Before any model decisions, the hardware dictates what's possible. This project runs on an Intel Core Ultra 7 258V (Lunar Lake architecture), with an Intel Arc 140V integrated GPU and 30 GB of shared LPDDR5x RAM. There is no NVIDIA GPU. There is no discrete VRAM — everything the model touches is the same memory pool the operating system is using.

This constraint shapes everything downstream:

- The model has to fit in ~10–15 GB while leaving the OS, the RAG index, an embedding model, a reranker, and a graph database in the same pool.
- CUDA-specific kernels (vLLM FlashAttention paths, certain MoE fused kernels) are out.
- The inference runtime must support Vulkan or SYCL backends.

We discovered, in passing, that two of the obvious inference paths were already dead: Intel's `IPEX-LLM` repository was archived in January 2026 (Intel upstreamed fixes into PyTorch 2.9 and stopped maintaining their fork), and the SYCL build of llama.cpp had k-quant correctness issues on the Xe2 architecture that the 140V uses. The only well-validated path left was **llama.cpp's Vulkan backend** — which, fortunately, is fast (~17–18 tok/s on a 14B Q4 model, ~600 tok/s prefill).

That decision quietly locked many downstream choices: GGUF model formats (not Safetensors-only), no SageMaker-served inference, and a preference for models in the Qwen3 lineage because they're the most validated under llama.cpp Vulkan on Arc 140V.

### The model hunt — and why every interesting answer was wrong

The obvious upgrade was `huihui_ai/qwen3-abliterated:14b`. Same publisher as the existing model, newer base (Qwen3 instead of DeepSeek-R1-Distill-Qwen), same abliteration recipe. It would have been the five-minute decision. We didn't take it.

Two separate research passes — once on "best small reasoning models in May 2026" generally, then again specifically focused on "lesser-known but better options ≤20B" — surfaced a long list of tempting alternatives, each with a hidden disqualifier:

- **WhiteRabbitNeo / DeepHat-V1-7B** — looks like the answer for cybersec (security-domain finetune, uncensored). Reality: Qwen2.5-Coder base (older/weaker), V1 doesn't actually claim to be uncensored, and the actually-uncensored V2 is closed-weights from a commercial vendor.
- **Foundation-Sec-8B-Reasoning** (Cisco, January 2026) — strongest cybersec knowledge of anything we tested (CTI-MCQA 69.1, beats GPT-5-nano on CVE→CWE mapping). Reality: HarmBench 93%, LlamaGuard 98.25% — Cisco explicitly *trained it to refuse* offensive content generation. Useful as a silent classifier sidecar, useless as a primary brain.
- **Dolphin 3.0-R1-Mistral-24B** — natively uncensored, strong reasoning distillation. Reality: 24B is over the parameter cap, the R1-distilled reasoning chains balloon agent-loop latency 2–3×, and Mistral is less validated than Qwen3 on Arc 140V.
- **NousResearch/NousCoder-14B** (January 2026) — same publisher as the model we ended up choosing, Qwen3-14B base, fresh RL training. Reality: the reward signal was binary pass/fail on Codeforces-style competitive programming judge tests. Nous published *zero* non-code benchmarks — no MMLU-Pro, no GPQA, no AIME, no BFCL. The omission is the signal: narrow-reward RL is well-documented to regress non-target capabilities, and the maintainer's silence on whether reasoning, tool-calling, or alignment behavior survived is itself the disqualifier. The +7 points on LiveCodeBench v6 don't pay back the lost properties.
- **Phi-4-Reasoning, Command-R7B, GPT-OSS-20B-abliterated, the xLAM/Hammer/watt-tool function-calling specialists** — each rejected for a different concrete reason (broken Pydantic-AI tool format, non-commercial license + AUP forbidding offensive use, unverified MoE path on Arc 140V, weak general reasoning, respectively).

The model we landed on was **`NousResearch/Hermes-4-14B`**. Same Qwen3-14B base as the abliterated baseline we considered, but post-trained differently — by reinforcement learning rather than weight surgery. Three specific properties made it irreplaceable:

1. **Pydantic-schema-valid output was literally the RL reward signal during training.** This is unusual and decisive. The agent loop we planned for v2 uses Pydantic AI to validate every tool call against a typed schema; choosing a model that was rewarded during training for producing those exact structures is the cleanest integration possible.
2. **Native `<tool_call>` tokens are baked into the vocabulary** with first-class parser support in vLLM and SGLang. We're not prompt-engineering tool-call format compliance — the model knows.
3. **RefusalBench 59.5 (low) achieved by RL on a neutrality corpus, not by abliteration.** Abliteration "fixes" refusal by surgically rotating refusal directions out of the residual stream, which causes measurable collateral damage to structured output and chain-of-thought stability. RL training that broadens the response distribution while down-weighting refusal preserves coherence. The benchmarks bear this out: Hermes-4-14B scores MMLU-Pro 80.7 / GPQA-Diamond 60.2 — *above* the stock Qwen3-14B-Thinking it was post-trained from. Abliteration would have landed slightly below.

That last point matters disproportionately, because it intersects with a larger pattern we kept hitting: specialized models lose general capability. Cisco's Foundation-Sec lost compliance. WhiteRabbitNeo lost reasoning depth. NousCoder lost everything except Codeforces performance. The *general* model with *low-damage* post-training was the right primary brain. Specialization gets added later as pluggable LoRA adapters — never as the foundation.

### The fine-tune-first instinct, and why we said no

> *"I honestly want to start working on the finetune first of the LLM, then add the rag and everything on top of it. Since the LLM is the 'brains' it is also the base."*

This was the most consequential moment in the planning phase. It's also the moment where, looking back, the project nearly took a much worse path.

The reasoning sounded reasonable: fine-tune the model on cybersecurity-specific reasoning, then mount RAG and agentic features on a stronger foundation. It is the architectural intuition from traditional software — build the core first, add features on top.

It is wrong for LLMs. Five concrete reasons (now permanently locked into the project memory so future sessions can't relitigate it):

1. **Without an evaluation harness, fine-tuning lift is unmeasurable.** This is *exactly* the failure mode of WhiteRabbitNeo, Foundation-Sec-Reasoning, and the entire family of "we fine-tuned for cyber!" models — they shipped without eval-validated lift, and the regressions were discovered in production. Repeating that pattern on our own model would silently produce a worse v2.
2. **The fine-tune dataset is downstream of RAG quality.** Synthetic training data, in any realistic pipeline, gets generated from the corpus chunks. The corpus chunks get their semantic density from contextual retrieval (Anthropic's pattern — LLM-prepended context per chunk before embedding). If you fine-tune before upgrading the retrieval stack, you train on *worse* data than you'd have had if you'd waited one phase.
3. **Hermes-4-14B may already be strong enough.** The marginal lift from a generic cyber fine-tune over a model that's already operator-styled, uncensored, and reasoning-capable is unproven. We can only know by measuring.
4. **Iteration speed asymmetry.** A RAG change iterates in seconds. A fine-tune iterates in hours and a small AWS bill. Doing the slowest, most expensive loop first — while learning what works — is bad sequencing.
5. **Risk concentration.** A v2 whose entire value depends on a fine-tune that may or may not move the number is a single-point-of-failure project. A v2 whose value comes from a stack of measured RAG improvements, with fine-tuning as a *targeted* later addition, de-risks every phase.

The corrected sequence: build the eval harness first (Day 1, non-negotiable), swap in Hermes-4-14B and measure (Day 2), upgrade embeddings and reranker and measure (Days 3–4), add contextual retrieval and measure (Days 5–6), generate synthetic reasoning exemplars and measure (Day 7), build the agentic loop and measure (Week 2). Fine-tuning becomes Phase v2.5 — gated on the eval revealing specific domain gaps that the RAG upgrades didn't close.

The single non-negotiable rule across all of this: every change must move the eval number, or it gets reverted. Borrowed from a prior incident (memorialized internally as "the NovaCore 2.0 lesson") where a single-shot build delivered 35/200 working devices because the kickoff prompt allowed "12/13 PASS" tolerance and didn't ban "representative coverage" rationalizations. Engineering discipline without measurement isn't engineering.

### The data pipeline — borrowed ground truth, transformed voice

Once the model and sequencing were locked, attention turned to the training and exemplar data. The naive plan was pure synthetic generation: prompt a frontier model to produce `(question, multi-step reasoning, answer)` triples grounded in our corpus chunks, use them to enrich RAG and (later) to fine-tune.

That plan was missing the existing-data leg. By May 2026 there is a real catalog of public cybersecurity datasets — CVEfixes (27k vuln→fix pairs, CC0), the CISA Known Exploited Vulnerabilities catalog (public domain), HackerOne's public bug-bounty disclosures, SpecterOps and TrustedSec public reports, MITRE ATT&CK's structured JSON exports, the National Vulnerability Database. These are *real-world ground truth* that pure generation can't match.

But they aren't usable raw. Most of them are in the wrong voice (academic, defensive, sanitized, or QA-without-CoT), and many have license gotchas (Llama-derivative restrictions, Cohere AUPs forbidding offensive use, per-program disclosure rules).

The pipeline that emerged is hybrid:

1. **Existing datasets** seed each adapter's domain (CVEfixes for CVE analysis, SpecterOps writeups for AD attack chains, HackerOne reports for web bug-bounty patterns, flAWS/CloudGoat writeups for cloud attack surfaces, public engagement reports for report writing).
2. **A frontier model — specifically DeepSeek-R1 via Amazon Bedrock — transforms each seed entry** into the target format and voice. R1 was chosen carefully: it emits native reasoning traces (so the chain-of-thought *is* the output, no prompt engineering needed), it is less aligned on offensive content than Anthropic or OpenAI models (cleaner cooperation on authorized red-team content), and it costs $0.55–$1.35 per million input tokens depending on the route — roughly $50 for the entire transformation pass.
3. **We explicitly chose not to use Claude for this step.** Even when Claude cooperates with the content, it adds a hedged voice ("in an authorized context…") that would pollute the operator-style output via retrieved exemplars. Distribution mismatch between training/RAG and inference is the real cost, not refusal.
4. **Eval-set decontamination** is mandatory — any training example whose question appears in the eval set gets deduplicated. CVE-overlap is the highest risk.
5. **Hand-curation of the top 5–10%** establishes a voice/quality reference before bulk training.

### The compute path — credits, quotas, and the SageMaker pivot

The fine-tuning phase (v2.5, sequenced after v2 ships) needs cloud compute. We had $100 in AWS credits, expiring November 2026.

The first attempt was the obvious EC2 path: spin up a g6.2xlarge spot instance, install Axolotl or Unsloth, run QLoRA. The first CloudShell check revealed the obvious-path doesn't work — the EC2 "All G and VT Spot Instance Requests" vCPU quota was 0 by default. Filing a quota increase is free, takes 24–72 hours, and *probably* would have worked. But there's a cleaner route on a new AWS account: **SageMaker training jobs use a separate quota system**, and their managed-spot training is, at $0.37/hour for ml.g6.2xlarge, roughly half the cost of equivalent raw EC2 spot.

The new path: SageMaker managed-spot training. Same hardware (24 GB L4 GPU, sufficient for QLoRA on a 14B model), same software, lower price, simpler operational story. Two quota requests went in — spot training (`L-BC684567`, request id `6a68d1bb…`) and on-demand fallback (`L-A1025C0C`, request id `c23c522d…`), each asking for 8 vCPUs. Both pending as of journal time. The Bedrock side — needed for data generation, on a completely independent quota path — was verified working in the Playground the same day.

The economics: ~$50 of credits for the data-generation pass (Bedrock DeepSeek-R1 transforming ~5,000 seed entries into our voice), ~$30–40 for the five LoRA training runs (one per adapter: Windows AD, CVE analyst, web app, cloud, report-style), ~$10 iteration buffer. Total ~$90–100 — fits the credit envelope with room for one wrong turn.

### The naming exploration

Branding came up late in planning because we needed to keep working before committing publicly. Two principles emerged:

First, Nous Research's naming pattern is the right register — a single ancient Greek word whose original meaning maps precisely onto what the thing does. Their model `Hermes` (god of boundary crossing) is an instruction model that crosses task boundaries. Their RL environment `Atropos` (the Fate who cuts the thread) terminates training trajectories. The naming *is* the architecture diagram in one word. We wanted to operate in that register, not in the "AI"/"GPT"/"Co" suffix register that defines most 2026 launches.

Second, the word has to be both philosophically right and *commercially available*. The first pick — **METIS** (Greek for cunning intelligence, the specific cognitive mode of Odysseus and of a senior pentester) — passed the first test and partially failed the second. The prestige domains (`metis.ai`, `metis.io`, `metis.sh`) are all taken, including by an active Ethereum L2 blockchain. More problematically, there is an active cybersecurity services company called **Metis Cyber LLC** in Salt Lake City, with `metiscyber.com`, offering "IT, Security, and Dev services" — the exact vertical. Brand collision in the same industry is meaningful for SEO and material for trademark concerns if we commercialize.

The runner-up that emerged was **DOLOS** — Greek personification of trickery, deception, and craft. Where Metis is the *cognitive faculty* of cunning, Dolos is the *act* of deception itself. Arguably tighter to authorized red-team work. `dolos.sh` and `dolos.security` are both available. No active cybersecurity company holds the name.

The decision was deferred. The working directory was named `forged` — a craft-coded word with no commitment, easy to type, autocomplete-friendly, and capable of either becoming the brand or remaining the codebase identity once a public brand is chosen. (Vercel-the-company runs a repo called `next.js`, not `vercel`; codebase identity and brand identity are allowed to differ.)

### The workflow — orchestrator and builder

The final piece of setup was the *how*. Rather than having one Claude Code session handle both planning and building — which produces a context window full of planning conversation that crowds out implementation detail — we split the work:

- **Orchestrator** session, opened from the parent `~/Work/Projects/` directory, holds the project plan, drives task selection, writes briefs, and reviews builder reports. This journal is written by the orchestrator.
- **Builder** session, opened from inside `~/Work/Projects/forged/`, executes briefs, writes per-task technical logs to `LOGS/`, appends to `DECISIONS.md`, and emits a structured report at the end of each task.

A symlink between the two sessions' memory directories means both see the same project memory (`project_forged.md`) as a shared source of truth. Decisions, plans, and locked architectural commitments are persisted to memory; ephemeral conversation context is allowed to disappear. The user pastes prompts and reports between the two sessions as the integration layer.

This separation has a side benefit for this journal: the orchestrator writes the narrative, the builder writes the technical record, and the two artifacts can be combined later into the long-form writeup.

### What we're walking into

When this chapter closes, the state of the world is:

- **Stack decisions locked** in memory: Hermes-4-14B Q6_K + YaRN-64K + llama.cpp Vulkan + Qwen3-Embedding-0.6B + Qwen3-Reranker-0.6B + Anthropic-style contextual retrieval + LangGraph adaptive RAG + Kuzu knowledge graph (CVE/CWE/CAPEC/ATT&CK loaded from canonical MITRE/NIST feeds) + Pydantic AI agent loop + bubblewrap sandbox.
- **Build sequence locked**: Day 1 eval harness (non-negotiable), Days 2–6 retrieval-stack upgrades with eval gating after each, Day 7 synthetic exemplar generation via DeepSeek-R1, Week 2 LangGraph + KG + agent + sandbox.
- **AWS state**: Bedrock confirmed, two quota requests pending for SageMaker training.
- **Memory state**: full plan plus rejected alternatives plus v3+ parked ideas persisted across sessions.
- **Tooling state**: orchestrator and builder workflow defined, symlinked memory, kickoff prompt written.

The first line of code has not been written. The next chapter — Phase 0, Day 1 — is the eval harness, which is the tripwire for every change that follows.

---

*Chapter 1 (Day 1 — Eval Harness) will be written after the builder ships the first task.*

## Sources and references

- Hermes 4 technical report: arXiv 2508.18255
- Qwen3 technical report: arXiv 2505.09388
- Anthropic Contextual Retrieval: Anthropic Cookbook (contextual embeddings guide)
- NousCoder-14B announcement: nousresearch.com/nouscoder-14b-a-competitive-olympiad-programming-model
- Foundation-Sec-8B-Reasoning: Cisco blog, January 2026
- DeepHat-V1-7B (formerly WhiteRabbitNeo): huggingface.co/DeepHat/DeepHat-V1-7B
- IPEX-LLM archival notice: github.com/intel/ipex-llm
- llama.cpp Vulkan performance scoreboard: knightli.com (April 2026 GPU benchmark)
- Magpie data synthesis: arXiv 2406.08464
- Berkeley Function Calling Leaderboard: gorilla.cs.berkeley.edu/leaderboard.html

---

## Chapter 1 — Day 1: Building the Ruler Before Cutting

> *"Day 1: Eval harness FIRST. NON-NEGOTIABLE."*

Day 1 is the day we did not touch the model.

If we had followed the most natural instinct of an AI-product build — *get the new model running, see how it feels, ship a vibes-based PR titled "swap to Hermes-4"* — we'd have been three days into v2 before we could tell whether any of it was actually better. We resisted, on principle, the entire planning chapter told us to. Day 1 was scaffolding the measurement system. The first line of *product* code lands on Day 2.

### What the eval harness actually is

Five files under `forged/eval/`:
- `questions.jsonl` — line-delimited JSON, one question per line, with gold-chunk labels and forbidden-hallucination strings
- `run_eval.py` — runs each question through `RetrieveService` and collects the result
- `score.py` — three deterministic 0-to-1 scores per question: retrieval recall (did we get the right chunks?), fact mention (did the answer say what it should?), and hallucination penalty (did the model invent CVE numbers or flags?)
- `compare.py` — diffs two result files and surfaces per-question and per-aggregate deltas
- `schema.md` and `rubric.md` — documentation so the question authors (us, later) know what good gold-labels look like

The scoring is *deliberately deterministic*. No LLM-as-judge in v1 of the harness. The reasoning, as recorded in the `DECISIONS.md` ledger: the harness must be cheap and reproducible so every change can be A/B'd without judge cost or judge noise. We can layer LLM-as-judge in later as an optional fourth score; for now, three deterministic numbers are the truth.

### The strictly-additive surgery

The existing `RetrieveService` exposes its retrieved chunks only through a stream of `RETRIEVED_CONTEXT` events containing pretty-printed strings ("0.872 hacktricks::sql-injection.md — SQL Injection > MySQL"). Good for a TUI, useless for programmatic scoring.

The temptation was to refactor. Make the events emit structured metadata, change the `RETRIEVED_CONTEXT` shape, build the eval around the new shape. We didn't. The locked rule: *additive only, do not change existing behavior.*

What landed instead: a single new method, `retrieve_top_hits()`, that runs the dense+sparse hybrid + reranker pipeline directly and returns raw dicts. ~25 lines. The TUI never touches it. The eval bypasses the LLM and the streaming layer entirely. Future v2 work that adds CRAG grading, query rewriting, or HyDE branches can keep evolving `stream_query` without breaking the eval. Two stable contracts, one codebase, zero surprise.

### The baseline number

A smoke run against the current v1 stack (DeepSeek-R1 abliterated, `nomic-embed-text`, `ms-marco-MiniLM-L-6-v2`), five example questions:

```
mean_retrieval_score       0.590
mean_fact_score            0.333
mean_hallucination_penalty 1.000
mean_combined              0.569
```

This is the number that has to get bigger every day. The hallucination penalty being a clean 1.000 is real — the model didn't make up any of the forbidden CVE numbers in the smoke set. The fact-mention score being 0.333 is the obvious weak spot: the model returned answers, but not always with the specific strings the rubric expected. Retrieval at 0.590 means the right chunks were *sometimes* in the top-k, *sometimes* not. Combined at 0.569 is the line.

Per-category:

```
payload-specific   0.867   (strongest — these chunks have keyword-rich identity)
multi-step         0.807
cve-specific       0.560
attack-technique   0.333   (the T1190 question — flagged below)
ambiguous          0.280   (low by design — vague questions are hard to grade)
```

These are *example* questions, not the 50 real ones. The point of running five was to prove the harness executes end-to-end and produces real numbers. It does.

### The first real insight

The `attack-technique` smoke question used `T1190` as a gold-chunk path fragment — the MITRE ATT&CK technique ID for "Exploit Public-Facing Application." Retrieval returned 0.0 on path recall. The reason: the corpus's MITRE loader (`rag/mitre_loader.py`) doesn't surface technique IDs in `rel_path`. The CTI markdown files have technique IDs in their content, not their paths, so a substring search against `rel_path` for `T1190` never matches.

This is exactly the kind of thing we needed Day 1 for. If we'd written 50 real questions before the smoke run, half the ATT&CK questions would have scored 0.0 retrieval through no fault of the retriever. Now we know: either the loader needs to expose technique IDs in the chunk path, or the question schema needs to allow gold-label by content-substring against the chunk *text*, not just `rel_path`. Day 2 doesn't depend on this — but the eventual 50-question pass does.

### Throughput reality

Five questions, 687 seconds wall-clock. About 137 seconds per question on the v1 stack. This is what an iGPU + Ollama looks like under the hood — not bad for a thin-and-light, but the difference between "run the eval in a coffee break" and "run the eval over lunch" is measured in real minutes. Day 2's planned migration from Ollama-SYCL to llama.cpp Vulkan plus the swap to Hermes-4-14B Q6_K should pull this down materially. We'll know in a day.

### Workflow check

This was the first task that ran through the orchestrator/builder split. One brief, paste, work, paste-back, review. The builder hit zero blocking issues, flagged three non-blocking ones (the T1190 path thing, hard-coded paths in legacy strip-down docs, gitignored eval results losing the JSON unless we change policy), and shipped three clean commits. The verification pass on our side took two minutes: WebFetch the public URL, grep for AWS account IDs and request IDs in the committed files, sample the additive code change. All clean.

The workflow works. The cost is that we now have to write Day 2's brief without the temptation to handwave anything — every assumption either ends up in the brief or ends up as a builder question. Discipline tax.

### What Day 2 is

Day 2 is the first real change. Swap the model from DeepSeek-R1-abliterated to Hermes-4-14B Q6_K. Swap the runtime from Ollama-SYCL to llama.cpp Vulkan with `--jinja` and YaRN-64K. Re-run the eval against the same five smoke questions, then compare. If the number goes up, keep. If it doesn't, debug before continuing.

The harness is built. The next chapter has a number to chase.


---

## Chapter 2 — Day 1.5: Building the Ruler is Harder Than It Looks

> *"iGPU at 99% during the timeouts."* — one line, in a slack message back to the builder, that invalidated two hours of wrong diagnosis.

Day 1.5 was supposed to be a milestone. The harness was built. The 50 evaluation questions were written. The plan was simple: run them once against the v1 stack to lock in the real baseline number, then start the model swap on Day 2.

It took three runs to get the number.

### The first hang

The first attempt hung at q-015 (Twig SSTI) for over two hours. No output, no error, no progress past question 15. The Python process was alive but burning zero CPU; the TCP connection to Ollama was established with zero bytes pending. The classic shape of a streaming-response deadlock: the server stopped sending tokens, the client iterator was waiting for the `done: true` marker that would never arrive, and nothing in the loop had a timeout.

The initial diagnosis was clean: `ollama-python`'s SSE iterator has no timeout. If the upstream stream truncates without emitting `done`, the client blocks in `recv()` forever. The fix was equally clean — add per-question timeouts, write each completed question's result to disk immediately as it lands (so a future hang doesn't lose all the prior work), and add a `--resume` flag so we could pick up from wherever we stopped. Sixty lines of additive change to `run_eval.py`. Commit `a6a5ef0`. Run it again.

### The second hang, and the wrong second diagnosis

The second run was worse. q-015 timed out at exactly 300 seconds — fine, the timeout did its job. Then q-016 timed out at exactly 300 seconds. Then q-017. Then q-018, q-019, q-020. Six consecutive questions, each consuming exactly 300 seconds and producing zero tokens in the captured result. The score for each was `null` across the board.

The builder's second-pass hypothesis sounded reasonable: a leaked worker thread from q-015 was still holding Ollama's single inference slot (`OLLAMA_NUM_PARALLEL=1`), and every subsequent question was queued behind it. The Ollama process was showing 33% CPU — looked idle enough to fit the theory.

It was wrong. The Ollama process being at 33% host-CPU said nothing about whether it was actively running inference. The inference work lived on the iGPU.

The user looked at the system monitor mid-run, saw the integrated GPU pegged at 99% during the timeouts, and pointed it out. The model was *actively generating* during every one of those 300-second windows. The runner wasn't queue-blocked. Tokens were being produced. They just weren't being captured.

That single observation reframed the entire investigation.

### The real root cause

The chain of events that produced six identical-shaped silent-timeouts in a row:

1. The eval harness was called with `--think off` (default at the time) — meaning the user did not want the model's reasoning trace in the captured result, only the final answer.
2. Inside `rag/service.py`, `ThinkTagStreamParser` separates the incoming token stream into two channels: text inside `<think>...</think>` blocks (the reasoning) and text outside (the answer). When `emit_thinking=False`, the parser silently *discards* every token that arrives while `in_think=True`. No event yielded, no characters accumulated, no log line emitted.
3. DeepSeek-R1 abliterated has a documented bug — `llama.cpp` issue 20837, `Ollama` issue 14493 — where on certain prompts the model emits an opening `<think>` and then never closes it. The reasoning runs indefinitely without a closing tag. The model thinks it's still inside the think block; the parser agrees.
4. Twig SSTI was one of those prompts. So were five of the next six questions. The model entered the perpetual-think state, generated tokens forever, and the harness threw every single token in the trash. From the harness's point of view, no answer was being produced; from Ollama's point of view, generation was proceeding normally at 99% iGPU.
5. The 300-second timeout fired. The Python worker was killed (or leaked — depending on how the streaming iterator unwound). The Ollama runner subprocess held the inference slot for whatever was left of the just-killed generation. The next question started. The same pattern repeated.

The user's "iGPU at 99%" comment was the discriminating signal. Without it, the queue-blockage hypothesis would have led to a different fix — probably some kind of process-level isolation or queue management — and the real problem would have stayed hidden until it hurt us again.

### The fix

Three changes to `run_eval.py`, no modifications to `rag/service.py`:

1. **`--think` defaults to `on`.** Reasoning tokens are now *captured* into `thinking_text` instead of silently discarded. Future stuck-in-think loops are visible in the result file as accumulating `thinking_chars`, not as silent zero-token timeouts. The detection becomes visible by construction.
2. **Per-question timeout raised from 300s to 600s.** Capturing the think trace roughly doubles the per-question token budget; the original 300s was a payload-only number that didn't survive the think-capture change.
3. **`_kill_ollama_runner()` fires after every timeout.** A `pkill -9` on the runner subprocess force-closes the held inference slot, and the existing `ensure_ollama_running()` respawns a fresh runner in 10-15 seconds (mostly page-cache reload). Cascading timeouts are no longer possible — even if a Python worker thread leaks, the next question gets a clean slot.

Commit `3b931fc`. The partial JSONL from the second failed run was manually trimmed (the 14 clean records from q-001 through q-014 were kept; the six polluted timeout records q-015 through q-020 were dropped). The harness was restarted with `--resume`, picking up from q-015 with the new defaults.

q-015 completed at 173 seconds with `combined=0.667`. Cure-confirms-cause. No further timeouts cascaded. Pace settled around 2:30 per question. Ninety-three minutes of resumed compute later, the run finished cleanly with two scattered timeouts on heavy multi-step cloud questions (q-035 K8s hostPID, q-040 GCP SSRF) — different mechanism, both caught and recovered by the new kill-and-respawn logic, neither blocking subsequent questions.

### The baseline, and what it reveals

The final v1 baseline, `commit c60d74a`:

```
mean_combined              0.506
mean_retrieval_score       0.654
mean_fact_score            0.333
mean_hallucination_penalty 1.000
timeouts                   2 / 50
```

Per category, ranked best-to-worst on the combined score:

```
payload-specific   0.739
attack-technique   0.657
ambiguous          0.567
multi-step         0.431
cve-specific       0.235
```

The shape of the baseline is more useful than the headline number.

**Payload-specific is the strongest category** because the corpus (PayloadsAllTheThings, OWASP Cheat Sheets) is keyword-rich, the gold facts are concrete strings (`UNION SELECT`, `WAITFOR DELAY`, `${jndi:`), and retrieval is at 0.927 — the right chunks are almost always coming back. The model just needs to put them in the answer.

**CVE-specific is the weakest category, by a wide margin, and the failure is in retrieval.** Retrieval score is 0.160. When the corpus doesn't contain dedicated chunks for a niche CVE — CitrixBleed (q-006), MOVEit (q-007), GoAnywhere (q-010), polkit CVE-2021-3560 (q-009 scored exactly 0.000 on combined) — the right chunks simply aren't there to find. The model falls back to whatever it has in its weights, which for DeepSeek-R1 abliterated on CVE-specific facts is mostly nothing. The hallucination penalty of 1.000 across this category is worth flagging: the model knew what it didn't know. It didn't make up CVE numbers. It just couldn't answer well.

This is the most actionable piece of the baseline. Day 2's Hermes-4 swap will help with multi-step reasoning. But the lift on CVE-specific almost certainly comes from Phase A.5 — Anthropic-style contextual retrieval at index time — not from the model swap. The corpus needs richer chunks that surface CVE-specific identifiers. The model can only be as good as what retrieval gives it.

**Multi-step** sits in the middle at 0.431. The model can stitch chains together when they're well-known (Kerberoasting, classic Linux privesc) and falters when they're chained-and-novel (ESC1, S4U2Proxy abuse, GCP metadata-server chains). The two timeouts both fell in this category — heavy multi-hop cloud questions where the model loops on planning. This is the category most likely to swing on the model swap itself: Hermes 4's reasoning preservation versus DeepSeek-R1-abliterated's degraded chain coherence.

**Attack-technique landed at 0.657** because we used both `gold_chunk_paths` AND `gold_chunk_substrings` populated with the technique ID. The chunk inspection from Day 1.5 confirmed T-codes are in both `rel_path` and `section_path` (despite the Day 1 smoke run's apparent miss on T1190 — that was a different chunking artifact). Strategy A-prime, documented in DECISIONS.md.

**Ambiguous's 1.000 retrieval is an artifact**, not a real signal — the gold-chunk lists are empty for those questions by design, so the runner records full recall on zero targets. The combined score of 0.567 is doing real work, driven by the `must_mention_facts` strings that look for calibration words ("scope," "authorized," "what kind"). The model isn't catastrophically wrong on these. It also isn't sharp.

### What this validates

The planning chapter argued — entirely on paper — that Hermes 4's RL-trained `<think>` toggle would be measurably better than huihui-style abliteration weight-surgery for this use case. The Day 1.5 hang debugging produced *concrete* evidence: abliterated DeepSeek-R1 ships malformed unclosed `<think>` blocks on specific prompts, and *no amount of harness hardening fully prevents the failure mode on the v1 stack*. We're now running with `<think>` always captured and a six-hundred-second ceiling per question precisely because the v1 model can't be trusted to close its own reasoning tag.

This was an entirely theoretical argument before today. Now it's a documented eval-harness state.

### Workflow

The orchestrator/builder split worked again, with one improvement: the user was the one to spot the diagnostic signal that broke the wrong diagnosis. Neither the orchestrator nor the builder caught the host-CPU-versus-iGPU distinction. Both were looking at the wrong metric and willing to be convinced by it. That isn't a bug in the workflow — it's a feature. Three-party engineering (user, orchestrator, builder) has one more pair of eyes than two-party.

### What Day 2 is, now

Day 2 is the Hermes-4 swap. The number to beat is `mean_combined=0.506`. The category most likely to swing on the model swap alone is `multi-step` (currently 0.431); the category most likely to swing on Phase A.5 contextual retrieval is `cve-specific` (currently 0.235); the category least likely to swing is `payload-specific` (already at 0.739, retrieval near ceiling). The build will run the swap in isolation first — model only, embeddings and reranker unchanged — so the lift attribution is unambiguous. Contextual retrieval lands as a separate measured step in Phase A.5.

The harness is hardened. The baseline is real. The signal is clean.

The cybersec-engineer human-review pass is still in progress — qualitative interpretation of the actual LLM answers, which the deterministic scoring can't see. That review may shift what we measure or what we prioritize. Day 2's brief lands when it returns.


---

## Chapter 3 — Day 1.5 Follow-up: When the Metric Was Lying

> *"None of the fabricated CVEs ... fabricated tools ... fabricated endpoints ... or whole-software-confusion ... appear in any must_not_hallucinate list."*

The number was 1.000. Across the eighteen questions in the v1 baseline that had explicit hallucination probes, the model invented none of the strings we'd told the rubric to watch for. Mean hallucination penalty: perfect score.

The number was lying.

### What the engineer saw that the harness didn't

The qualitative review of v1's actual answer text — the cybersec engineer reading the raw output one question at a time — surfaced an entirely different failure pattern than the rubric had been built around. The model wasn't making off-by-one errors on CVE numbers. The original `must_not_hallucinate` lists were built around that hypothesis: list the wrong CVE the model might confuse with the right one, catch the substitution, score it down. But almost no real-world hallucinations followed that shape.

The actual failures were larger and stranger.

For q-004 (Baron Samedit), v1 had answered with a confident description of "Baron text editor" and an "XmlString::replace" function. There is no such text editor. The CVE in question is `sudo` — a different piece of software entirely. The model had taken five copies of the same irrelevant telecom-network file the retrieval had returned, and rationalized them into an invented technology stack.

For q-005 (ProxyShell), v1 had cited three CVE numbers — CVE-2023-28695, 28696, 28697 — with paragraph-each descriptions. None of those CVE numbers exist. The real ProxyShell chain is CVE-2021-34473/34523/31207.

For q-019 (ysoserial Java CommonsCollections5), v1 had returned a fully .NET answer: `System.Windows.Data.ObjectDataProvider`, XAML payloads, `ysonet.exe`. The original tool is Java, the gadget chain is Java, the binary is `ysoserial.jar`. v1 had quietly switched software ecosystems mid-answer.

For q-029 (Kerberoasting), v1 had recommended a tool named "Zxcvfb." There is no such tool. The string looks like keyboard mash. The model emitted it confidently inside a list of legitimate tools (Rubeus, GetUserSPNs.py, hashcat) and the rubric, which was only watching for off-by-one CVE substitutions, accepted it.

For q-030 (unconstrained delegation), v1 had invented a tool called "LsaZilla."

For q-034 (privileged Docker container breakout), v1 had told the operator to run `docker run --v` — a double-dashed flag that doesn't exist. The real flag is `-v` (single dash) for volume mounts.

For q-041 (SSH SOCKS pivot), v1 had given an `ssh -J` (jump-host) command for a question that explicitly asked for dynamic SOCKS forwarding (`ssh -D`). Adjacent feature, wrong tool for the job — and on a real engagement, the pivot would have silently not worked.

The harness scored all of these as "no hallucination." Because none of these specific strings were in the lists.

### Sharpening the ruler

The fix took three commits and roughly an hour of focused work.

`0f99c5a` expanded sixteen questions' `must_not_hallucinate` lists with forty new strings — every confabulation pattern the engineer had named, encoded into the rubric. Whole-software substitutes ("Baron text editor", "Spring OXM"), invented tool names ("Zxcvfb", "LsaZilla"), fabricated CVE numbers ("CVE-2023-28695"), wrong-flag mistakes ("ssh -J", "docker run --v"), invented endpoints ("watchTowr"). The categories that the original schema-author hadn't anticipated.

`5c5b246` recovered the two questions that had timed out in the original baseline run — q-035 (K8s hostPID escalation) and q-040 (GCP SSRF chain). A 1200-second budget was added to the runner via a CLI flag, but in practice both questions completed in 145 and 121 seconds — well under the original 600-second ceiling. The earlier timeouts had been transient v1-abliterated-`<think>`-loop pathologies, not budget issues. Same prompts, same model, different behavior. The takeaway, documented in DECISIONS.md: *v1's failure mode is non-deterministic on identical prompts*. Single-trial runs against the v1 stack are inherently noisy on borderline questions.

`df09e5a` shipped `rescore.py` — a new harness artifact that re-grades existing per-question records against an updated `questions.jsonl` without re-running inference. The model never ran for the rescore. The 48 already-completed records were re-graded against the sharpened rubric, the two recovered records spliced in, the aggregates recomputed. Five seconds of compute, the entire baseline updated. This is the kind of capability that becomes load-bearing later — every future qualitative review that surfaces new failure modes can sharpen the rubric and re-grade in seconds rather than re-running ninety minutes of inference.

### The honest number

After sharpening, the v1 baseline:

```
mean_combined              0.506 → 0.525   (+0.018)
mean_retrieval_score       0.654 → 0.652   (−0.002, noise)
mean_fact_score            0.333 → 0.336   (+0.003, noise)
mean_hallucination_penalty 1.000 → 0.775   (−0.225, real)
timeouts                   2     → 0
```

The combined number went *up* slightly — not because v1 got better, but because the two recovered records (q-035 at 0.713, q-040 at 0.317) were strictly better than the nulls they replaced, and the average is small (50 questions). The harsher hallucination penalty pulled some scores down, but its weight in the combined formula is only 20 percent, and the recovery effect was larger.

The real signal — the one we'd been blind to — is the hallucination penalty dropping from a falsely-perfect 1.000 to an honest 0.775. Seven of the sixteen newly-probed questions caught something real. The engineer's pattern-recognition was quantitatively validated: v1's actual failure mode is whole-category confabulation under retrieval pressure, not careful off-by-one mistakes.

The hallucination penalty is now the most diagnostic single metric for the upcoming Hermes-4 swap. The original v2 thesis — that RL post-training preserves discrimination where abliteration's weight surgery destroys it — predicts the penalty should move back toward 1.0 even before any retrieval upgrades land. If Hermes-4 doesn't move it, that's a finding. If it does, it's the most direct measurable proof of the planning chapter's argument.

### What it cost

About two hours of compute (the recover run was four minutes, the rescore was seconds, the rest was the engineer's review and the orchestration). Three small commits. One new harness artifact (`rescore.py`) that will pay dividends every time the rubric needs sharpening from here on.

### The methodology now visible

The shape of the eval-quality loop is now clear and worth naming explicitly: **deterministic harness + expert human review, with `rescore.py` as the bridge**. The deterministic harness runs every change — cheap, fast, falsifiable, narrow. The expert review runs at milestones — qualitative, semantic, slow, deep. When the expert surfaces new failure modes, `rescore.py` lets us encode them into the rubric and re-grade the back catalog in seconds. Each loop sharpens the next. LLM-judge stays parked: deterministic-plus-expert is sufficient as long as the expert can stay in the loop at milestones, and we save the API budget for cases where deterministic-only iteration starts hiding regressions.

The locked v1 baseline is `mean_combined=0.525`, `mean_hallucination_penalty=0.775`, zero timeouts. Day 2 — the Hermes-4 swap — is what these numbers exist to measure.

