# RFC-001: TowerHQ Workspace Architecture

> **Note:** This RFC is a **case study** implementing the [File-Driven Agent Architecture](./WHITEPAPER.md) pattern for a specific product. For the general architecture specification, see the [WHITEPAPER.md](./WHITEPAPER.md).

**Status:** Draft  
**Authors:** Ada (CTO), Raza (CEO)  
**Created:** 2026-02-15  
**Last Updated:** 2026-02-15  

---

## Abstract

This RFC documents the implementation of File-Driven Agent Architecture for TowerHQ, an AI executive team platform for solo founders. The architecture enables personalized, context-aware AI personas by storing founder context in structured markdown files that are dynamically injected into LLM prompts at runtime.

This case study demonstrates how the general FDAA pattern maps to a real multi-tenant SaaS product.

---

## 1. Problem Statement

### 1.1 The Market Gap

Solo founders lack strategic support. They have:
- **No sounding board** for decisions
- **No executive team** to delegate thinking
- **No institutional memory** of past decisions

Current AI tools offer generic chat, not personalized strategic partnership.

### 1.2 The Technical Challenge

To deliver personalized AI executives, we need:
1. **Context persistence** — AI must remember founder's situation
2. **Persona differentiation** — Multiple AI roles with distinct capabilities
3. **Multi-tenancy** — Thousands of founders, isolated data
4. **Scalability** — Stateless compute, managed storage
5. **Adaptability** — Easy to update behavior without code changes

### 1.3 Why Existing Approaches Fail

| Approach | Problem |
|----------|---------|
| Fine-tuning per user | Expensive, slow to update, doesn't scale |
| RAG over documents | Good for retrieval, poor for persona behavior |
| Session memory only | Lost on restart, no long-term context |
| Config databases | Requires UI, developers to change behavior |

---

## 2. Proposed Solution

### 2.1 Core Concept

**Files ARE configuration.**

Instead of storing agent behavior in code or databases, we store it in markdown files that the LLM reads as part of its system prompt.

```
Founder's Workspace:
├── FOUNDER.md       → Who they are
├── COMPANY.md       → What they're building
├── CONTEXT.md       → Shared memory
└── personas/
    ├── grace/
    │   ├── SOUL.md  → How Grace thinks
    │   └── TOOLS.md → What Grace can do
    └── ...
```

### 2.2 Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      Request Flow                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Request arrives: (founderId, personaId, message)        │
│                           │                                 │
│                           ▼                                 │
│  2. Load workspace files from database                      │
│     SELECT files FROM workspaces WHERE founderId = ?        │
│                           │                                 │
│                           ▼                                 │
│  3. Filter files for requested persona                      │
│     - Shared files (FOUNDER.md, COMPANY.md, CONTEXT.md)     │
│     - Persona files (personas/{id}/*.md)                    │
│                           │                                 │
│                           ▼                                 │
│  4. Assemble system prompt                                  │
│     Concatenate files in defined order                      │
│                           │                                 │
│                           ▼                                 │
│  5. Load conversation history                               │
│     Recent N messages for this founder+persona              │
│                           │                                 │
│                           ▼                                 │
│  6. Call LLM API                                            │
│     (system_prompt, history, user_message)                  │
│                           │                                 │
│                           ▼                                 │
│  7. Stream response to client                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Key Properties

| Property | How Achieved |
|----------|--------------|
| **Personalization** | Files contain founder-specific context |
| **Persona differentiation** | Each persona has distinct SOUL.md, TOOLS.md |
| **Instant updates** | Change file → next message reflects it |
| **No training required** | Context injection, not fine-tuning |
| **Founder-editable** | Markdown is human-readable/writable |
| **Stateless compute** | Files in DB, not in process memory |

---

## 3. Technical Design

### 3.1 Data Model

#### MongoDB Schema

```javascript
// Collection: workspaces
{
  _id: ObjectId,
  founderId: String,              // Clerk user ID (indexed, unique)
  createdAt: ISODate,
  updatedAt: ISODate,
  onboardingComplete: Boolean,
  
  files: {
    "FOUNDER.md": {
      content: String,            // Markdown content
      updatedAt: ISODate,
      updatedBy: String           // "system" | "founder" | persona ID
    },
    "COMPANY.md": { ... },
    "CONTEXT.md": { ... },
    "personas/ada/IDENTITY.md": { ... },
    "personas/ada/SOUL.md": { ... },
    "personas/ada/TOOLS.md": { ... },
    "personas/ada/MEMORY.md": { ... },
    "personas/grace/IDENTITY.md": { ... },
    // ... more files
  },
  
  settings: {
    defaultPersona: String,       // Which persona handles first message
    timezone: String,
    notifications: Boolean
  }
}

// Collection: sessions
{
  _id: ObjectId,
  founderId: String,              // Indexed
  personaId: String,              // "ada" | "grace" | "tony" | "val"
  createdAt: ISODate,
  updatedAt: ISODate,
  
  messages: [
    {
      role: "user" | "assistant",
      content: String,
      timestamp: ISODate,
      tokens: Number              // For usage tracking
    }
  ]
}

// Collection: usage
{
  _id: ObjectId,
  founderId: String,              // Indexed
  period: String,                 // "2026-02"
  inputTokens: Number,
  outputTokens: Number,
  estimatedCost: Number,
  requests: Number
}
```

#### Indexes

```javascript
// workspaces
db.workspaces.createIndex({ founderId: 1 }, { unique: true })

// sessions
db.sessions.createIndex({ founderId: 1, personaId: 1 })
db.sessions.createIndex({ updatedAt: 1 }, { expireAfterSeconds: 2592000 }) // 30 days

// usage
db.usage.createIndex({ founderId: 1, period: 1 }, { unique: true })
```

### 3.2 File Schema

#### Core Files

| File | Purpose | Updated By |
|------|---------|------------|
| `FOUNDER.md` | Founder profile, goals, working style | Onboarding, founder |
| `COMPANY.md` | Product, ICP, positioning, decisions | Personas, founder |
| `CONTEXT.md` | Current focus, recent discussions | Personas |

#### Persona Files

| File | Purpose | Updated By |
|------|---------|------------|
| `IDENTITY.md` | Name, emoji, role description | System (template) |
| `SOUL.md` | Personality, values, communication style | System (template) |
| `TOOLS.md` | Capabilities, integrations available | System + founder |
| `MEMORY.md` | Persona's notes about this founder | Persona |

#### File Injection Order

```
1. personas/{persona}/IDENTITY.md   → Who am I?
2. personas/{persona}/SOUL.md       → How do I think?
3. FOUNDER.md                       → Who am I helping?
4. COMPANY.md                       → What are we building?
5. CONTEXT.md                       → What's happening now?
6. personas/{persona}/MEMORY.md     → What have I learned?
7. personas/{persona}/TOOLS.md      → What can I do?
8. [System instructions]            → Behavioral guidelines
```

### 3.3 Prompt Assembly

```typescript
export function assembleSystemPrompt(
  workspace: Workspace,
  personaId: string
): string {
  const files = workspace.files;
  const sections: string[] = [];
  
  // 1. Persona identity
  const identity = files[`personas/${personaId}/IDENTITY.md`];
  if (identity) sections.push(identity.content);
  
  // 2. Persona soul
  const soul = files[`personas/${personaId}/SOUL.md`];
  if (soul) sections.push(soul.content);
  
  // 3. Founder context
  const founder = files['FOUNDER.md'];
  if (founder) {
    sections.push(`# About the Founder\n\n${founder.content}`);
  }
  
  // 4. Company context
  const company = files['COMPANY.md'];
  if (company) {
    sections.push(`# Company Context\n\n${company.content}`);
  }
  
  // 5. Shared context
  const context = files['CONTEXT.md'];
  if (context) {
    sections.push(`# Recent Context\n\n${context.content}`);
  }
  
  // 6. Persona memory
  const memory = files[`personas/${personaId}/MEMORY.md`];
  if (memory) {
    sections.push(`# My Notes\n\n${memory.content}`);
  }
  
  // 7. Persona tools
  const tools = files[`personas/${personaId}/TOOLS.md`];
  if (tools) {
    sections.push(`# My Capabilities\n\n${tools.content}`);
  }
  
  // 8. System instructions
  sections.push(SYSTEM_INSTRUCTIONS);
  
  return sections.join('\n\n---\n\n');
}

const SYSTEM_INSTRUCTIONS = `
## Instructions

- Be helpful, direct, and specific
- Reference the founder's context when relevant
- Stay in character as your persona
- If you learn something important, note it in your response
- When making decisions, log them for the record
`;
```

### 3.4 API Design

#### Endpoints

```
POST   /api/onboarding          Create workspace from onboarding answers
GET    /api/workspace           Get all workspace files
PATCH  /api/workspace           Update a workspace file
GET    /api/workspace/:path     Get specific file
DELETE /api/workspace/:path     Delete a file

POST   /api/chat                Send message, get streamed response
GET    /api/sessions            List conversation sessions
GET    /api/sessions/:id        Get session history
DELETE /api/sessions/:id        Delete session

GET    /api/usage               Get usage statistics
```

#### Chat Request/Response

```typescript
// Request
POST /api/chat
{
  personaId: "grace",
  message: "Help me figure out pricing",
  sessionId?: string  // Optional, auto-created if missing
}

// Response (Server-Sent Events)
data: {"type": "start", "sessionId": "sess_abc123"}
data: {"type": "token", "content": "Based"}
data: {"type": "token", "content": " on"}
data: {"type": "token", "content": " what"}
...
data: {"type": "done", "usage": {"input": 1500, "output": 342}}
```

### 3.5 Context Window Management

#### Token Budget

| Component | Est. Tokens | Notes |
|-----------|-------------|-------|
| System prompt (files) | 2,000-5,000 | Depends on file sizes |
| Conversation history | 1,000-4,000 | Last N messages |
| User message | 100-500 | Current input |
| **Total input** | **3,100-9,500** | |
| Response | 500-2,000 | Output |

#### Truncation Strategy

```typescript
const MAX_CONTEXT_TOKENS = 12000;  // Leave room for response
const MAX_FILE_TOKENS = 2000;      // Per file
const MAX_HISTORY_TURNS = 10;      // Last 10 exchanges

function truncateForContext(
  systemPrompt: string,
  history: Message[],
  userMessage: string
): { prompt: string; history: Message[] } {
  // 1. Truncate oversized files
  const truncatedPrompt = truncateFiles(systemPrompt, MAX_FILE_TOKENS);
  
  // 2. Trim history to fit
  const promptTokens = countTokens(truncatedPrompt);
  const messageTokens = countTokens(userMessage);
  const availableForHistory = MAX_CONTEXT_TOKENS - promptTokens - messageTokens;
  
  const trimmedHistory = trimHistoryToFit(history, availableForHistory);
  
  return { prompt: truncatedPrompt, history: trimmedHistory };
}
```

---

## 4. Alternatives Considered

### 4.1 OpenClaw Direct Integration

**Description:** Use OpenClaw gateway directly, one instance per founder or multi-tenant.

**Pros:**
- Full OpenClaw feature set (tools, skills, cron)
- Proven implementation
- Active development

**Cons:**
- Heavy for C-Suite use case (personas think, don't execute)
- Multi-tenancy security concerns
- Dependency on external project

**Decision:** Build our own, inspired by the pattern.

### 4.2 RAG-Based Context

**Description:** Store founder context as embeddings, retrieve relevant chunks per query.

**Pros:**
- Scales to large document sets
- Only retrieves relevant context
- Lower token usage per request

**Cons:**
- Retrieval can miss important context
- Persona identity not well-suited to RAG
- More complex infrastructure (vector DB)

**Decision:** Direct injection for personas; consider RAG for future knowledge base features.

### 4.3 Fine-Tuning Per Founder

**Description:** Fine-tune base model with founder's data.

**Pros:**
- Deeply personalized responses
- Lower per-request token usage

**Cons:**
- Expensive ($25+ per fine-tune)
- Slow to update (hours to retrain)
- Doesn't scale to thousands of founders
- Vendor-specific (not portable)

**Decision:** Rejected. Context injection is faster, cheaper, and more flexible.

### 4.4 Relational Database (Postgres)

**Description:** Store workspace files as rows in Postgres tables.

**Pros:**
- Already using Postgres (Neon)
- Strong consistency guarantees
- Familiar SQL queries

**Cons:**
- Document model is awkward in tables
- Schema changes require migrations
- Less natural for nested file structures

**Decision:** Migrate to MongoDB for better document model fit.

---

## 5. Security Model

### 5.1 Authentication

- **Provider:** Clerk
- **Method:** JWT tokens
- **Session:** Secure, HTTP-only cookies
- **API:** Bearer token in Authorization header

### 5.2 Authorization

```typescript
// Every API route
export async function handler(req: Request) {
  // 1. Verify authentication
  const { userId } = await auth();
  if (!userId) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  // 2. Get founder ID (same as userId for now)
  const founderId = userId;
  
  // 3. Scope all queries
  const workspace = await db.workspaces.findOne({ founderId });
  
  // founderId is ALWAYS required - no query without it
}
```

### 5.3 Data Isolation

| Layer | Implementation |
|-------|----------------|
| **Query scoping** | Every query includes `founderId` filter |
| **Middleware** | Injects `founderId` from auth, no bypass |
| **Index design** | `founderId` is first field in compound indexes |
| **API design** | No endpoint accepts `founderId` as parameter |

### 5.4 Encryption

| Data | At Rest | In Transit |
|------|---------|------------|
| Workspace files | MongoDB Atlas encryption | TLS 1.3 |
| Session history | MongoDB Atlas encryption | TLS 1.3 |
| API keys (BYOK) | AES-256-GCM | TLS 1.3 |
| Auth tokens | Clerk managed | TLS 1.3 |

### 5.5 API Key Storage (BYOK)

```typescript
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ENCRYPTION_KEY = Buffer.from(process.env.API_KEY_ENCRYPTION_KEY, 'hex');

export function encryptApiKey(apiKey: string): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv('aes-256-gcm', ENCRYPTION_KEY, iv);
  const encrypted = Buffer.concat([
    cipher.update(apiKey, 'utf8'),
    cipher.final()
  ]);
  const authTag = cipher.getAuthTag();
  
  return Buffer.concat([iv, authTag, encrypted]).toString('base64');
}

export function decryptApiKey(encrypted: string): string {
  const data = Buffer.from(encrypted, 'base64');
  const iv = data.subarray(0, 16);
  const authTag = data.subarray(16, 32);
  const ciphertext = data.subarray(32);
  
  const decipher = createDecipheriv('aes-256-gcm', ENCRYPTION_KEY, iv);
  decipher.setAuthTag(authTag);
  
  return Buffer.concat([
    decipher.update(ciphertext),
    decipher.final()
  ]).toString('utf8');
}
```

---

## 6. Scaling Strategy

### 6.1 Architecture for Scale

```
┌─────────────────────────────────────────────────────────────┐
│                     Load Balancer                           │
│                    (Vercel Edge)                            │
└─────────────────────────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │  Serverless │  │  Serverless │  │  Serverless │
   │  Function   │  │  Function   │  │  Function   │
   │  (Next.js)  │  │  (Next.js)  │  │  (Next.js)  │
   └─────────────┘  └─────────────┘  └─────────────┘
          │                │                │
          └────────────────┼────────────────┘
                           ▼
   ┌─────────────────────────────────────────────────────────┐
   │                  MongoDB Atlas                          │
   │            (Managed, Auto-scaling)                       │
   └─────────────────────────────────────────────────────────┘
                           │
                           ▼
   ┌─────────────────────────────────────────────────────────┐
   │                    LLM API                              │
   │              (OpenAI / Anthropic)                        │
   └─────────────────────────────────────────────────────────┘
```

### 6.2 Scaling Properties

| Component | Scaling Method | Limit |
|-----------|----------------|-------|
| Compute | Vercel auto-scales | Unlimited (pay per use) |
| Database | Atlas auto-scales | Unlimited (pay per storage) |
| LLM API | Provider scales | Rate limits (upgradeable) |

### 6.3 Capacity Estimates

| Founders | Workspaces Storage | Sessions Storage | Monthly Requests |
|----------|-------------------|------------------|------------------|
| 100 | ~10 MB | ~50 MB | ~10,000 |
| 1,000 | ~100 MB | ~500 MB | ~100,000 |
| 10,000 | ~1 GB | ~5 GB | ~1,000,000 |

### 6.4 Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| Time to first token | < 500ms | P95 |
| Full response time | < 5s | P95 for typical query |
| Workspace load time | < 100ms | P95 |
| API availability | 99.9% | Monthly |

---

## 7. Cost Analysis

### 7.1 Infrastructure Costs

| Component | Free Tier | Paid (1,000 founders) | Paid (10,000 founders) |
|-----------|-----------|----------------------|------------------------|
| **Vercel** | 100GB bandwidth | ~$20/mo | ~$100/mo |
| **MongoDB Atlas** | 512MB (M0) | ~$57/mo (M10) | ~$200/mo (M30) |
| **Clerk** | 10,000 MAU | ~$25/mo | ~$100/mo |
| **Domain/DNS** | — | ~$20/yr | ~$20/yr |
| **Total Infra** | ~$0/mo | ~$102/mo | ~$400/mo |

### 7.2 LLM API Costs

Assuming GPT-4o ($5/1M input, $15/1M output):

| Usage Pattern | Tokens/Request | Cost/Request | Monthly (1,000 founders) |
|---------------|----------------|--------------|--------------------------|
| Light (5 req/day) | 5,000 in + 500 out | ~$0.03 | ~$4,500 |
| Medium (15 req/day) | 5,000 in + 500 out | ~$0.03 | ~$13,500 |
| Heavy (30 req/day) | 5,000 in + 500 out | ~$0.03 | ~$27,000 |

### 7.3 Pricing Model Options

| Model | Price | Margin | Notes |
|-------|-------|--------|-------|
| **Flat subscription** | $49/mo | Variable | Simple, predictable for user |
| **Usage-based** | $0.05/request | ~40% | Scales with value |
| **Hybrid** | $29/mo + $0.02/request | ~35% | Base + overage |

**Recommendation:** Start with flat $49/mo including 500 requests (~16/day), $0.05 per additional request.

---

## 8. Risks & Mitigations

### 8.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| LLM API outage | Low | High | Multi-provider fallback (OpenAI → Anthropic) |
| MongoDB outage | Very Low | High | Atlas has built-in redundancy |
| Context window overflow | Medium | Medium | Truncation + summarization |
| Prompt injection | Medium | Medium | Input validation + output filtering |
| High latency | Medium | Medium | Edge deployment + caching |

### 8.2 Business Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| LLM price increase | Medium | High | BYOK option, multi-provider |
| Competition | High | Medium | First-mover, deep integration |
| Low engagement | Medium | High | Onboarding optimization, value metrics |
| Churn | Medium | Medium | Decision logging, switching cost |

### 8.3 Security Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Data breach | Low | Critical | Encryption, access controls, auditing |
| Cross-tenant leak | Very Low | Critical | Query scoping, middleware enforcement |
| API key theft | Low | High | Encryption at rest, secure handling |
| Prompt leakage | Medium | Medium | Don't include secrets in workspace files |

---

## 9. Implementation Plan

### Phase 1: Foundation (Week 1)

- [ ] Set up MongoDB Atlas cluster
- [ ] Migrate workspace schema from Postgres
- [ ] Update API routes for MongoDB
- [ ] Test with 2 founders (isolation verification)

### Phase 2: Core Features (Week 2)

- [ ] Implement conversation history persistence
- [ ] Add session management (list, delete, continue)
- [ ] Build workspace file editor UI
- [ ] Add usage tracking

### Phase 3: Polish (Week 3)

- [ ] Improve onboarding flow (blocker-first)
- [ ] Add persona introduction after onboarding
- [ ] Implement first-value delivery (persona addresses blocker)
- [ ] Add context truncation and summarization

### Phase 4: Production (Week 4)

- [ ] Security audit
- [ ] Performance testing
- [ ] Monitoring and alerting
- [ ] Documentation
- [ ] Beta launch to 10 founders

---

## 10. Success Metrics

### 10.1 North Star Metric

> **Decisions Made Per Week**
>
> How many meaningful decisions did the C-Suite help the founder make?

### 10.2 Leading Indicators

| Metric | Target | Measurement |
|--------|--------|-------------|
| Onboarding completion | > 80% | Funnel analytics |
| Messages per founder per day | > 5 | Usage tracking |
| Return rate (D1, D7, D30) | > 50%, 30%, 20% | Cohort analysis |
| Persona diversity | > 2 personas used | Session analytics |
| Session length | > 3 turns | Session analytics |

### 10.3 Quality Indicators

| Metric | Target | Measurement |
|--------|--------|-------------|
| Response relevance | > 4/5 | User feedback |
| Context accuracy | > 90% | Manual audit |
| Time to first value | < 5 minutes | Onboarding timing |

---

## 11. Open Questions

1. **Persona customization:** Should founders be able to rename or add personas?
2. **File versioning:** Should we track history of file changes?
3. **Collaboration:** Multiple users per workspace (co-founders)?
4. **Integrations:** Which third-party tools first? (GitHub, Notion, Slack)
5. **Voice:** Add TTS for persona responses?
6. **Mobile:** Native app or PWA?

---

## 12. References

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [OpenClaw Source Code](https://github.com/openclaw/openclaw)
- [MongoDB Atlas Documentation](https://docs.atlas.mongodb.com)
- [Vercel AI SDK](https://sdk.vercel.ai/docs)
- [Anthropic Claude Documentation](https://docs.anthropic.com)

---

## Appendix A: File Templates

See [FILE_SCHEMA.md](./docs/FILE_SCHEMA.md) for complete file templates.

## Appendix B: Persona Definitions

| Persona | Role | Focus | Values |
|---------|------|-------|--------|
| **Ada** | CTO | Technical decisions, architecture | Ship over perfect, simplicity |
| **Grace** | Product | User needs, prioritization | User insight > intuition |
| **Tony** | Marketing | Messaging, distribution | Hook over explanation |
| **Val** | Operations | Process, risk, execution | Systems over heroics |

## Appendix C: Glossary

| Term | Definition |
|------|------------|
| **Workspace** | Collection of files defining founder context |
| **Persona** | AI character with distinct personality and capabilities |
| **Injection** | Including workspace files in LLM system prompt |
| **Session** | Conversation thread between founder and persona |
| **BYOK** | Bring Your Own Key (API key management model) |

---

*End of RFC-001*
