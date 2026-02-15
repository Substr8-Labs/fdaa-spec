# TowerHQ Architecture

> AI Executive Team for Solo Founders — Architecture Documentation

## Overview

TowerHQ provides founders with an AI C-Suite (Ada, Grace, Tony, Val) that understands their context, remembers decisions, and delivers real value from day one.

This repo documents the architecture decisions and technical design.

## Quick Links

- [Architecture Overview](./docs/ARCHITECTURE.md)
- [File Schema](./docs/FILE_SCHEMA.md)
- [Onboarding Flow](./docs/ONBOARDING.md)
- [Decision Log](./docs/DECISIONS.md)
- [Prototype Spec](./docs/PROTOTYPE.md)

## Core Insight

Inspired by OpenClaw's pattern:

> **Files ARE configuration.**
> 
> Markdown files define who the AI is, what it knows, and how it behaves.
> Change a file → change the agent. No restart, no retrain, no config UI.

## Architecture Summary

```
┌─────────────────────────────────────────────┐
│              TowerHQ Gateway                │
├─────────────────────────────────────────────┤
│  Request: (founderId, personaId, message)   │
│                    ↓                        │
│  Router: Load workspace for founder         │
│                    ↓                        │
│  Assembler: Inject files into prompt        │
│                    ↓                        │
│  LLM: Call with assembled context           │
│                    ↓                        │
│  Response: Stream back to founder           │
└─────────────────────────────────────────────┘
```

## Tech Stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Frontend | Next.js (Vercel) | Already deployed, serverless |
| Database | MongoDB Atlas | Document model fits workspaces |
| Auth | Clerk | Already integrated |
| LLM | OpenAI / Anthropic | Platform key, BYOK later |
| Storage | MongoDB documents | Files as JSON fields |

## Key Patterns

### 1. Workspace as Document
Each founder has a workspace containing all their context files.

### 2. Dynamic Prompt Injection
Files are read and injected into the system prompt on every request.

### 3. Persona Isolation
Each persona (Ada, Grace, etc.) has their own files (SOUL.md, TOOLS.md).

### 4. Stateless Compute
Gateway is stateless. All state lives in the database. Horizontal scaling is trivial.

## Status

- [x] Architecture designed
- [x] File schema defined
- [x] Onboarding flow designed
- [x] Postgres prototype built
- [ ] MongoDB migration
- [ ] Multi-founder testing
- [ ] Production deployment

---

Built by [Substr8 Labs](https://substr8labs.com)
