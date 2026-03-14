# Awesome ML Interview Notes

Interview-focused notes for:

- LLM systems
- RL and post-training
- agent / reasoning / tool use
- inference and serving optimization

This repo is built for fast revision before ML, LLM, applied research, and AI systems interviews.

## What Makes This Useful

Most interview notes fail in one of two ways:

- they are too shallow to survive real technical interviews
- they are too long and messy to review under time pressure

This repo is meant to sit in the middle:

- compact enough to skim quickly
- deep enough to explain system design, tradeoffs, and execution details

The notes are especially biased toward:

- LLM post-training
- inference and serving systems
- agents and reasoning
- interview-ready explanations instead of textbook-style coverage

## Who This Is For

- students preparing for ML / LLM interviews
- applied researchers preparing for systems or post-training interviews
- engineers who want a compact map of modern LLM topics

## Repository Map

### Core LLM

- [LLM Optimization Notes](notes/core-llm/llm-optimization-notes.md)
  Memory, compute, inference, KV cache, quantization, parallelism, and serving optimizations.
- [Speculative Decoding](notes/core-llm/speculative-decoding.md)
  Intuition, math, acceptance-rejection logic, residual distribution, and interview framing.

### RL and Post-Training

- [Reinforcement Learning](notes/rl/reinforcement-learning.md)
  MDP, value/policy methods, TD vs MC, Q-learning, PPO, actor-critic, and RLHF connections.
- [SFT, RLHF, DPO, RLAIF](notes/rl/sft-rlhf-dpo-rlaif.md)
  A compact map of modern LLM post-training pipelines and their tradeoffs.

### Agent and Reasoning

- [Tool Use](notes/agent-and-reasoning/tool-use.md)
  What tool use is, how it differs from function calling, and why it looks like sequential decision-making.
- [MCP, Function Calling, Agent Framework](notes/agent-and-reasoning/mcp-function-calling-agent-framework.md)
  A clean separation between protocol, action interface, and system orchestration.
- [Agent Core Components](notes/agent-and-reasoning/agent-core-components.md)
  Planning, memory, reflection, verifier, router, recovery, and long-horizon failure.
- [Reasoning, Planning, Search](notes/agent-and-reasoning/reasoning-planning-search.md)
  CoT, self-consistency, ToT, MCTS, reflection, and verifier-guided search.
- [Memory, Context Engineering, RAG](notes/agent-and-reasoning/memory-context-engineering-rag.md)
  How to separate long-term state, external retrieval, and context construction.
- [Evaluation, Verifier, Reward Model, Judge Model](notes/agent-and-reasoning/evaluation-verifier-reward-model-judge-model.md)
  A practical map of evaluation-time and training-time feedback components.
- [Reasoning Model Training](notes/agent-and-reasoning/reasoning-model-training.md)
  Process supervision, outcome supervision, PRM, ORM, best-of-N, and test-time scaling.
- [Math, Code, Agent Reasoning](notes/agent-and-reasoning/math-code-agent-reasoning.md)
  How verifier strength, reward shape, and task structure differ across these reasoning domains.

### Systems Walkthroughs

- [Mini-SGLang Walkthrough](notes/systems/mini-sglang-walkthrough.md)
  A detailed source walkthrough of Mini-SGLang from request lifecycle to scheduler, radix cache, paged KV, engine, and attention backend execution.

## Recommended Reading Paths

### Path 1: LLM Systems Interview

1. [LLM Optimization Notes](notes/core-llm/llm-optimization-notes.md)
2. [Speculative Decoding](notes/core-llm/speculative-decoding.md)
3. [Tool Use](notes/agent-and-reasoning/tool-use.md)
4. [Agent Core Components](notes/agent-and-reasoning/agent-core-components.md)
5. [Mini-SGLang Walkthrough](notes/systems/mini-sglang-walkthrough.md)

### Path 2: LLM Post-Training Interview

1. [Reinforcement Learning](notes/rl/reinforcement-learning.md)
2. [SFT, RLHF, DPO, RLAIF](notes/rl/sft-rlhf-dpo-rlaif.md)
3. [Evaluation, Verifier, Reward Model, Judge Model](notes/agent-and-reasoning/evaluation-verifier-reward-model-judge-model.md)
4. [Reasoning Model Training](notes/agent-and-reasoning/reasoning-model-training.md)

### Path 3: Agent / Reasoning Interview

1. [Tool Use](notes/agent-and-reasoning/tool-use.md)
2. [MCP, Function Calling, Agent Framework](notes/agent-and-reasoning/mcp-function-calling-agent-framework.md)
3. [Agent Core Components](notes/agent-and-reasoning/agent-core-components.md)
4. [Reasoning, Planning, Search](notes/agent-and-reasoning/reasoning-planning-search.md)
5. [Memory, Context Engineering, RAG](notes/agent-and-reasoning/memory-context-engineering-rag.md)
6. [Math, Code, Agent Reasoning](notes/agent-and-reasoning/math-code-agent-reasoning.md)

## Notes About The Notes

- Most notes are written in Chinese because they were optimized for fast interview revision.
- The repo is intentionally compact and high-signal rather than exhaustive.
- The Mini-SGLang walkthrough references the upstream `sgl-project/mini-sglang` repository rather than vendoring the source here.

## Suggested Positioning

If someone lands on this repo, the intended impression is:

- not a random notebook dump
- not a generic ML summary
- a compact, serious, interview-oriented LLM knowledge base

If you find it useful, star the repo so I know which topics to expand next.
