# Awesome ML Interview Notes

Curated interview-focused notes on:

- LLM systems
- RL and post-training
- Agents, reasoning, and tool use
- inference and optimization

This repo is designed for fast review before ML / LLM / AI systems interviews.

## Why This Repo

Most interview prep notes are either:

- too shallow to be useful in real interviews
- too long and unstructured to review quickly

This repo aims to sit in the middle:

- concise enough to revise fast
- deep enough to explain system design and modeling tradeoffs

## Structure

### `notes/core-llm`

- `speculative-decoding.md`
- `llm-optimization-notes.md`

### `notes/rl`

- `reinforcement-learning.md`
- `sft-rlhf-dpo-rlaif.md`

### `notes/agent-and-reasoning`

- `tool-use.md`
- `mcp-function-calling-agent-framework.md`
- `agent-core-components.md`
- `reasoning-planning-search.md`
- `memory-context-engineering-rag.md`
- `evaluation-verifier-reward-model-judge-model.md`
- `reasoning-model-training.md`
- `math-code-agent-reasoning.md`

### `notes/systems`

- `mini-sglang-walkthrough.md`

## Recommended Reading Order

If you are preparing for LLM systems / applied research interviews:

1. `notes/core-llm/llm-optimization-notes.md`
2. `notes/core-llm/speculative-decoding.md`
3. `notes/rl/reinforcement-learning.md`
4. `notes/rl/sft-rlhf-dpo-rlaif.md`
5. `notes/agent-and-reasoning/tool-use.md`
6. `notes/agent-and-reasoning/agent-core-components.md`
7. `notes/agent-and-reasoning/reasoning-model-training.md`
8. `notes/systems/mini-sglang-walkthrough.md`

## Notes

- Most notes are written in Chinese because they are optimized for fast interview revision.
- The Mini-SGLang walkthrough references the upstream `sgl-project/mini-sglang` repository rather than vendoring the source code here.

## Suggested Repo Name

If you want a public-facing name that is clear and searchable, use:

- `awesome-ml-interview-notes`

It is broad enough for ML interviews and still specific enough to be discoverable.
