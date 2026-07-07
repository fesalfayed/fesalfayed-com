---
title: "Fine-tuning gpt-oss-20b for tool calling, in four formats"
description: "A LoRA SFT of OpenAI's 20B MoE for agent-loop tool use, shipped as BF16 / 4-bit / MLX / GGUF. What worked, what the format matrix taught me, and what still needs real evals."
date: 2026-07-06
tags: [fine-tuning, gpt-oss, tool-use, agents, quantization]
toc: true
---

The artifact first: a tool-use fine-tune of OpenAI's `gpt-oss-20b`, shipped as a
[four-format family on Hugging Face](https://huggingface.co/collections/fesalfayed/finetuned-hermes-function-calling-v1)
— full BF16, MXFP4 4-bit, MLX for Apple Silicon, and GGUF in five quants for
llama.cpp/Ollama/LM Studio. As of late June the family had crossed **9,000
downloads**, with the GGUF and 4-bit variants doing most of the volume — which
tells you something real about where local-agent inference actually happens.

An honest framing note up front: this is a build log, not a benchmark victory
lap. The eval section below is mostly *planned*, and I'd rather show you the
status table than invent numbers. Posts on this site keep negative results and
open gaps in; that's the house rule.

## 1. Why fine-tune for tool calling at all

Agent frameworks live or die on one behavior: when the loop hands the model a
tool schema and says *call it*, the model must emit clean, parseable JSON — no
markdown fences, no "Sure! Here's the call you asked for", no arguing with the
system prompt. Base chat models are trained to be conversational; agent
harnesses need them to be *mechanical* exactly at the call boundary and
conversational everywhere else.

`gpt-oss-20b` was an attractive base: a 21B-parameter Mixture-of-Experts with
only ~3.6B active parameters per token, Apache-2.0, with a native reasoning-effort
knob in its Harmony chat format. Small enough to serve locally, smart enough to
plan multi-step tool chains.

The targets for the fine-tune:

- **Function-calling adherence** — correct JSON, no commentary mid-call
- **Long agent loops** — 10+ turns of tool → observe → plan without drifting
- **System-prompt fidelity** — role boundaries and allow-list rules hold

## 2. Training setup

- **Base:** `openai/gpt-oss-20b`
- **Method:** LoRA SFT, rank 64 / alpha 16, merged back into BF16
- **Frame:** Unsloth + TRL on a single H100 (80 GB)
- **Data:** ~42k tool-use traces from real agent sessions, filtered for
  successful tool calls and clean JSON — no synthetic distillation
- **Sequence length:** 8192 with packing
- **Loss:** assistant-only; user, system, and tool turns masked

The single most important lesson from this project isn't a hyperparameter —
it's a format bug. Read §5.

## 3. The format matrix, and what downloads reveal

One training run, four artifacts:

| Artifact | Runtime | Fits on | Role |
| :--- | :--- | :--- | :--- |
| BF16 (~41 GB) | vLLM, Transformers | A100/H100 class | reference checkpoint, further quantization |
| MXFP4 4-bit | Transformers | 16 GB GPU | low-VRAM experiments, Colab |
| MLX | mlx_lm | Apple Silicon | Mac-native local inference |
| GGUF (Q3_K_M → F16) | llama.cpp, Ollama, LM Studio | 16 GB RAM and up | the local-agent workhorse |

The download distribution inverted my expectations. The "real" BF16 checkpoint
— the thing an ML engineer would consider the model — is the least downloaded
by two orders of magnitude. The quantized consumer formats carry essentially
all the traffic. If you ship a fine-tune and stop at safetensors, you've
published it for approximately nobody: **the format matrix is not packaging,
it's distribution**.

Two serving notes that cost me time:

- MoE models are sensitive to sampler settings; `min_p: 0.1` noticeably
  stabilizes expert routing at temperature 0.7. Drop temperature to ~0.2 for
  strict tool-call turns.
- The Harmony reasoning-effort knob survives the fine-tune: `Reasoning: high`
  costs roughly 3–4× the output tokens and is measurably better on multi-step
  tool plans. `low` is right for single-call turns.

## 4. Evals — status, not vibes

What exists today:

| Eval | Status |
| :--- | :--- |
| JSON/tool-call syntax validity | planned — parseability check over generated call blocks |
| Single-tool selection | planned — exact-match tool choice on held-out prompts |
| Multi-turn tool replay | in progress — replaying real traces with tool results |
| Live agent integration | manual smoke tests in the harness that consumes the calls |

No number appears here until it has a reproducible script and a frozen input
file. The strongest claim I'll make without those: the fine-tune runs daily
inside a real agent loop on my own machines, and the failure mode it was built
to fix — commentary leaking into call JSON — is gone from those sessions.

## 5. What didn't work (and one bug worth your time)

**The chat-template mismatch.** The costliest failure of the project: an early
run fed Hermes-style function-calling data — ChatML-formatted, with XML-ish
`<tool_call>` blocks — through the Harmony chat template that `gpt-oss-20b`
actually speaks. The two formats disagree about where tool calls live and how
turns are delimited. The result wasn't a crash; it was a *quietly degraded*
model that mostly worked and subtly mangled call boundaries. Format bugs in
fine-tuning don't announce themselves — they tax you at inference forever. If
you take one thing from this post: **diff your rendered training examples
against the base model's native template before you spend a GPU-hour.**

Also cut or deferred:

- **Publishing the full training corpus.** The traces come from my own agent
  sessions; a small synthetic sample ships in the repo instead. Transparency
  loses to privacy here, and I'd rather say so than pretend.
- **Benchmark-suite numbers at launch.** Tool-use benchmarks that don't match
  your harness's calling convention measure the benchmark, not the agent.
  The replay eval in §4 is the honest version; it's still in progress.

## 6. Limitations

- Math and code generation are unchanged from base — this optimizes the agent
  loop, not raw reasoning.
- The model can over-call tools on vague instructions; a "if you can answer
  directly, do so" system line mitigates.
- English-only training mix. Not safety-tuned beyond the base.

## 7. References

- [openai/gpt-oss-20b](https://huggingface.co/openai/gpt-oss-20b) — base model (Apache-2.0)
- [Unsloth](https://github.com/unslothai/unsloth) + [TRL](https://github.com/huggingface/trl) — training frame
- [The model family on HF](https://huggingface.co/collections/fesalfayed/finetuned-hermes-function-calling-v1): [BF16](https://huggingface.co/fesalfayed/gpt-oss-20b-hermes_agent-tool-finetune_16bit) · [4-bit](https://huggingface.co/fesalfayed/gpt-oss-20b-hermes_agent-tool-finetune_4bit) · [MLX](https://huggingface.co/fesalfayed/gpt-oss-20b-hermes_agent-tool-finetune_mlx) · [GGUF](https://huggingface.co/fesalfayed/gpt-oss-20b-hermes_agent-tool-finetune_gguf)
- [Project card on GitHub](https://github.com/fesalfayed/gpt-oss-20b-hermes-tool-finetune) — export matrix, eval scripts as they land

*"Hermes-Agent" in the model names refers to the local agent framework these
models serve — not affiliated with NousResearch's Hermes model series.*
