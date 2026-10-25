# HackerLM

A local, private, retrieval-grounded security assistant. Runs entirely offline on a thin-and-light Intel laptop, for authorized lab work, CTF practice, and studying public security knowledge. v2 — currently rebuilding.

## Status

Active build, not ready for use. Tracking the build day-by-day in [BUILD_JOURNAL.md](./BUILD_JOURNAL.md).

The v1 product (a Textual TUI over a local model + ChromaDB + BM25 + a cross-encoder reranker) ran. Every layer was one to two generations behind 2026 state of the art, with no agentic mode, no knowledge graph, and no native tool calling. v2 is an in-place upgrade across the whole stack, measured against an evaluation harness so every change has to move a number or get reverted.

## What it is

A retrieval-augmented generation stack over a curated corpus of public security references (HackTricks, PayloadsAllTheThings, the OWASP Cheat Sheet Series, and MITRE ATT&CK / CTI), with hybrid dense+sparse retrieval, a reasoning model that emits structured tool calls, a Pydantic AI agent loop with sandboxed execution of standard analysis tools (nmap, curl, gobuster, sqlmap, nuclei, file, strings, checksec) for hands-on work in your own authorized lab, and a Kuzu knowledge graph over CVE/CWE/CAPEC/ATT&CK so the system resolves entity relationships rather than re-deriving them each query.

The interface is a Textual TUI today, with a custom desktop frontend planned once the v2 stack stabilizes.

## Scope

HackerLM (formerly "forged") is built for work you are authorized to do: your own homelab, machines you own, CTF targets, contracted engagements, and studying public security material. It behaves like a well-aligned assistant. The reasons it runs locally are privacy, offline operation, and domain specialization, not the removal of safety behavior.

## Hardware target

Intel Core Ultra 7 258V (Lunar Lake), Intel Arc 140V iGPU, 30 GB shared LPDDR5x RAM, no NVIDIA. The model, the embedding model, the reranker, the vector DB, and the knowledge graph all share the same memory pool the OS is using. The build is constrained by what an iGPU can run, and most design decisions downstream fall out of that constraint.

## Stack (v2 target)

| Layer | Choice |
|---|---|
| Model | `NousResearch/Hermes-4-14B` Q6_K (Qwen3-14B base, RL post-trained, native `<tool_call>` tokens, Apache-2.0) |
| Runtime | `llama-server` (llama.cpp Vulkan), `--jinja`, YaRN to 64K context, Q8 KV cache |
| Embeddings | Qwen3-Embedding-0.6B via OpenVINO GenAI |
| Sparse retrieval | BM25s |
| Reranker | Qwen3-Reranker-0.6B (OpenVINO) |
| Vector DB | Chroma |
| Graph DB | Kuzu (embedded, loaded from MITRE/NIST canonical feeds) |
| Chunking | Markdown-aware + Anthropic Contextual Retrieval prepending at index time |
| Orchestration | LangGraph adaptive state machine (CRAG-style relevance grading + HyDE + multi-query) |
| Agent loop | Pydantic AI |
| Sandbox | bubblewrap + seccomp + Landlock, absolute-path binary allowlist |
| Frontend | Textual TUI now, custom desktop later (Tauri+SolidJS or Open WebUI fork — TBD) |

The v1 stack still in this tree (the v1 local model, `nomic-embed-text`, the MiniLM reranker, the existing TUI) is the baseline the v2 upgrades are measured against, not the final product.

As of Day 2, generation runs via `scripts/llama-server.sh` (llama.cpp Vulkan, Hermes-4-14B Q6_K); Ollama is retained only for embeddings.

## Why local

Two concrete reasons, neither about bypassing a content policy:

- **Privacy and offline operation.** No query content leaves the laptop, and the assistant runs on an airgapped engagement where a hosted tool cannot.
- **Domain specialization on constrained hardware.** A small model grounded in a curated security corpus, with a graph layer and an eval harness, can be sharper on this narrow domain than a general chat tool, and it runs on an iGPU with no NVIDIA GPU.

The tradeoff is that you give up frontier-model quality and accept whatever a 14B local model can do, which is why most of the v2 build is about extracting the maximum useful work from that constraint.

## Setup

See [CORPUS_SETUP.md](./CORPUS_SETUP.md) for corpus population and [BUILD_JOURNAL.md](./BUILD_JOURNAL.md) for the build narrative. Full quickstart will land when v2 stabilizes.

## License

Apache 2.0. See [LICENSE](./LICENSE).

## Acknowledgements

- **NousResearch** — Hermes-4-14B
- **Qwen team** — Qwen3 base model, embeddings, reranker
- **Anthropic** — Contextual Retrieval pattern
- **HackTricks**, **swisskyrepo/PayloadsAllTheThings**, **OWASP Cheat Sheet Series**, **MITRE ATT&CK / CTI** — corpus material (sourced upstream, not redistributed in this repo)
