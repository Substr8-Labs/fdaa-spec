# File-Driven Agent Architecture

**A Pattern for Portable, Persistent, and Provable AI Agents**

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Substr8 Labs](https://img.shields.io/badge/Substr8-Labs-blue)](https://substr8labs.com)

---

## Overview

This repository contains the specification and research for **File-Driven Agent Architecture (FDAA)** — a design pattern where AI agents are defined entirely through human-readable markdown files.

> **The intelligence is in the model. The identity is in the files.**

Instead of embedding agent configuration in code, databases, or fine-tuned weights, FDAA stores all agent state in structured files that are dynamically injected into the LLM context at runtime.

## Why File-Driven?

| Current Approaches | File-Driven |
|-------------------|-------------|
| Config scattered across code, DBs, model weights | Everything in versioned files |
| Agent trapped in implementation | Copy files = move agent |
| Hard to audit what agent "knows" | Human-readable, diff-able |
| Fine-tuning: expensive, slow | Change file → instant update |
| Vendor lock-in | Works with any LLM |

## Research Validation

- **74% accuracy** on memory tasks (Letta benchmark) — outperforms RAG and specialized tools
- **28.64% faster** execution with structured markdown
- **90% cost reduction** via prompt caching
- **100% portability** across LLM providers

## Documents

### Core

- **[WHITEPAPER.md](./WHITEPAPER.md)** — Full specification and research (~9,000 words)

### Supporting

- [docs/FILE_SCHEMA.md](./docs/FILE_SCHEMA.md) — Workspace file structure
- [docs/DECISIONS.md](./docs/DECISIONS.md) — Design decision log
- [docs/PROTOTYPE.md](./docs/PROTOTYPE.md) — Implementation notes

### Case Study

- [RFC-001-WORKSPACE-ARCHITECTURE.md](./RFC-001-WORKSPACE-ARCHITECTURE.md) — TowerHQ implementation (AI C-Suite for founders)

## Quick Example

An agent is just a folder of markdown files:

```
my_agent/
├── IDENTITY.md       # Who the agent is
├── SOUL.md           # How it thinks
├── MEMORY.md         # What it remembers
├── TOOLS.md          # What it can do
└── CONTEXT.md        # Current state
```

Minimal implementation:

```python
from pathlib import Path
from openai import OpenAI

def load_workspace(path: str) -> str:
    files = sorted(Path(path).glob("*.md"))
    return "\n\n---\n\n".join(
        f"## {f.name}\n\n{f.read_text()}" for f in files
    )

def chat(workspace: str, message: str) -> str:
    client = OpenAI()
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": load_workspace(workspace)},
            {"role": "user", "content": message}
        ]
    )
    return response.choices[0].message.content
```

## Core Concepts

### 1. Files ARE Configuration

No config UI needed. Edit markdown → agent changes immediately.

### 2. Stateless Runtime

The gateway holds no state. Files provide context, LLM provides intelligence.

### 3. W^X Policy (Write XOR Execute)

Agents can update `MEMORY.md` but cannot modify `IDENTITY.md` or `SOUL.md`.

### 4. Tenant Isolation

Every query scoped by tenant ID. No cross-tenant access possible.

### 5. Prompt Caching

Stable file content = cacheable system prompts = 90% cost reduction.

## Scaling

| Agents | Storage | Monthly Cost |
|--------|---------|--------------|
| 100 | 10 MB | ~$0 (free tier) |
| 10,000 | 1 GB | ~$200 |
| 100,000 | 10 GB | ~$500 |

## Security

- **W^X enforcement** — Identity files are read-only
- **Tenant isolation** — Query scoping prevents cross-tenant access
- **Injection protection** — File content sanitized
- **Drift monitoring** — Detect behavioral deviation
- **Canary tasks** — Periodic integrity verification

## Future Directions

1. **Agent Interchange Format** — Standard manifest for agent portability
2. **Cryptographic Verification** — Signed, verifiable agent state
3. **Federated Agents** — Cross-organizational collaboration
4. **Self-Evolving Agents** — Specification improvement with human approval

## About Substr8 Labs

We build infrastructure for **provable AI agents** — systems that are transparent, auditable, and cryptographically verifiable.

- **Website:** https://substr8labs.com
- **GitHub:** https://github.com/Substr8-Labs
- **Twitter:** https://x.com/substr8labs

## License

This work is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

You are free to share and adapt this material for any purpose, with attribution.

---

*Built by [Substr8 Labs](https://substr8labs.com) — Provable Agent Infrastructure*
