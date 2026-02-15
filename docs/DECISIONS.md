# TowerHQ Decision Log

## 2026-02-15: Architecture Session

### Decision 1: Build vs Buy (OpenClaw)

**Context:** Should TowerHQ use OpenClaw directly, or build our own implementation inspired by it?

**Options Considered:**
| Option | Pros | Cons |
|--------|------|------|
| A: Build ourselves | Simple, scalable, control | Rebuild existing work |
| B: OpenClaw per founder | Full power | Expensive, complex |
| C: Multi-tenant OpenClaw | Efficient | Security harder |

**Decision:** **Option A (Build ourselves)**

**Rationale:**
- C-Suite personas mostly think, not execute
- API-based tools cover 80% of needs
- Easier to secure multi-tenant
- Faster to iterate
- Can graduate to Option C later if needed

---

### Decision 2: Database

**Context:** How to store workspace files?

**Options Considered:**
| Option | Pros | Cons |
|--------|------|------|
| Postgres (current) | Already have it | Not ideal for documents |
| MongoDB Atlas | Document model fits | Migration needed |
| S3 + metadata | Cheap, actual files | No queries |

**Decision:** **MongoDB Atlas**

**Rationale:**
- Workspace = document (natural fit)
- Single query gets entire workspace
- Free tier (512MB) is plenty
- Built-in vector search for future
- Schema-less = easy to evolve

---

### Decision 3: API Key Model

**Context:** How do users pay for LLM usage?

**Options Considered:**
| Model | Description | Fit |
|-------|-------------|-----|
| Platform pays | We have one key, bill users | Simple |
| BYOK | Users bring own key | No margin risk |
| Hybrid | Platform default, BYOK option | Best of both |

**Decision:** **Hybrid (start with Platform pays)**

**Rationale:**
- Platform pays: No onboarding friction
- Include X tokens in subscription
- Add BYOK later for enterprise
- Track usage per founder for billing

---

### Decision 4: File Schema

**Context:** What files define a workspace?

**Decision:** Custom schema inspired by OpenClaw

```
FOUNDER.md     — Who they are
COMPANY.md     — What they're building
CONTEXT.md     — Shared memory
personas/*/    — Per-persona files
```

**Rationale:**
- Semantic names (not generic like "config.md")
- Founder-friendly (they can read/edit)
- Persona isolation (each has own files)
- Shared context (some files cross personas)

---

### Decision 5: Onboarding Flow

**Context:** How do founders set up their workspace?

**Decision:** Discovery conversation → file generation

**Flow:**
1. "What's been stuck?" (capture blocker)
2. "What are you building?" (product context)
3. "Who's it for?" (ICP)
4. "What stage?" (lifecycle)
5. Generate workspace files
6. Introduce C-Suite with first value

**Rationale:**
- Blocker-first = immediate relevance
- Conversation, not form
- AI generates files (not manual config)
- First value in <5 minutes

---

### Decision 6: Scaling Model

**Context:** How does TowerHQ scale?

**Decision:** Stateless compute + managed database

```
Vercel (auto-scales)
    ↓
MongoDB Atlas (managed)
    ↓
OpenAI API (infinite scale)
```

**Rationale:**
- Gateway has no intelligence (just routing)
- LLM has all intelligence (API)
- Files have all context (database)
- Horizontal scaling is trivial

---

### Decision 7: Data Isolation

**Context:** How do we isolate founder data?

**Decision:** Query scoping + middleware enforcement

```typescript
// Every query requires founderId
const workspace = await db.workspaces.findOne({ founderId });

// Middleware enforces scoping
req.db = createScopedDb(auth.founderId);
```

**Rationale:**
- Simple and effective for MVP
- No cross-tenant leakage possible
- Can upgrade to separate collections later
- Enterprise: separate databases if needed

---

## Open Questions

1. **Persona customization:** Can founders rename/add personas?
2. **Skill marketplace:** How do we package/share capabilities?
3. **Voice output:** Add TTS for persona responses?
4. **Real-time updates:** WebSocket for live file edits?
