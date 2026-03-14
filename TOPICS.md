# Topics Map

This page is the shortest way to understand what this repository covers.

## LLM Systems

- [LLM Optimization Notes](notes/core-llm/llm-optimization-notes.md)
- [Speculative Decoding](notes/core-llm/speculative-decoding.md)
- [Mini-SGLang Walkthrough](notes/systems/mini-sglang-walkthrough.md)

Focus:

- KV cache
- attention backends
- speculative decoding
- paged KV
- scheduler design
- serving throughput / latency tradeoffs

## RL and Post-Training

- [Reinforcement Learning](notes/rl/reinforcement-learning.md)
- [SFT, RLHF, DPO, RLAIF](notes/rl/sft-rlhf-dpo-rlaif.md)

Focus:

- value vs policy methods
- PPO / actor-critic
- preference optimization
- reward modeling
- LLM post-training pipelines

## Agent and Reasoning

- [Tool Use](notes/agent-and-reasoning/tool-use.md)
- [MCP, Function Calling, Agent Framework](notes/agent-and-reasoning/mcp-function-calling-agent-framework.md)
- [Agent Core Components](notes/agent-and-reasoning/agent-core-components.md)
- [Reasoning, Planning, Search](notes/agent-and-reasoning/reasoning-planning-search.md)
- [Memory, Context Engineering, RAG](notes/agent-and-reasoning/memory-context-engineering-rag.md)
- [Evaluation, Verifier, Reward Model, Judge Model](notes/agent-and-reasoning/evaluation-verifier-reward-model-judge-model.md)
- [Reasoning Model Training](notes/agent-and-reasoning/reasoning-model-training.md)
- [Math, Code, Agent Reasoning](notes/agent-and-reasoning/math-code-agent-reasoning.md)

Focus:

- tool use and agents
- reasoning-time search
- memory and context construction
- verifier and judge systems
- process vs outcome supervision

## Best Starting Points

If you only read 5 notes, start with:

1. [LLM Optimization Notes](notes/core-llm/llm-optimization-notes.md)
2. [Speculative Decoding](notes/core-llm/speculative-decoding.md)
3. [Tool Use](notes/agent-and-reasoning/tool-use.md)
4. [Reinforcement Learning](notes/rl/reinforcement-learning.md)
5. [Mini-SGLang Walkthrough](notes/systems/mini-sglang-walkthrough.md)
