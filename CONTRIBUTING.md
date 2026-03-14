# Contributing

This repository is organized as a compact interview-focused ML knowledge base.

The goal is not to maximize note count. The goal is to keep the notes:

- high-signal
- interview-oriented
- technically defensible
- quick to review under time pressure

## What To Add

Good additions usually satisfy at least one of these:

- a high-frequency interview topic
- a concept people often confuse with nearby concepts
- a system design topic that benefits from implementation-level explanation
- a note that compresses a large topic into a reviewable form

## What To Avoid

- generic textbook restatements
- low-signal filler
- notes that are broad but shallow
- copying articles without restructuring them into interview notes

## Writing Style

- Prefer Chinese for long-form content if it makes revision faster.
- Keep titles and navigation English-friendly.
- Optimize for fast recall, not literary style.
- Prefer direct explanations over abstract phrasing.
- Include tradeoffs, failure modes, and "why this matters" whenever relevant.

## Recommended Structure For New Notes

Use [TEMPLATE.md](TEMPLATE.md) as the starting point.

A strong note usually contains:

- what the concept is
- why it matters
- how it works
- where it helps
- where it fails
- what nearby concepts it is often confused with
- how to explain it in an interview

## File Placement

- `notes/core-llm/`: inference, decoding, optimization, model systems
- `notes/rl/`: RL, post-training, reward modeling, preference optimization
- `notes/agent-and-reasoning/`: tool use, agents, reasoning, planning, memory, evaluators
- `notes/systems/`: walkthroughs, serving engines, system design, codebase analysis

## Tone Standard

Aim for notes that feel like:

- serious revision material
- compact but deep
- useful before interviews

Avoid notes that feel like:

- blog post fluff
- notebook dumps
- copied documentation

## Linking

When a new note overlaps existing notes:

- add it to the relevant section `README.md`
- add it to [TOPICS.md](TOPICS.md) if it becomes a key topic
- add it to [README.md](README.md) if it materially changes the repository entry points
