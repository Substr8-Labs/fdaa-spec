# TowerHQ Architecture

## The Core Pattern

TowerHQ follows the OpenClaw pattern: **files define the agent**.

Instead of configuration UIs, databases of settings, or fine-tuned models, we use markdown files that get injected into the LLM's system prompt on every request.

```
Every request:
1. Load workspace files from database
2. Assemble into system prompt
3. Call LLM with prompt + conversation
4. Return response

The LLM has no memory. The files ARE the memory.
```

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        TowerHQ                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│   │   Vercel    │    │   MongoDB   │    │   OpenAI    │    │
│   │   (Next.js) │    │   Atlas     │    │   API       │    │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    │
│          │                  │                  │            │
│          │    ┌─────────────┴─────────────┐    │            │
│          └────┤      Request Flow         ├────┘            │
│               └───────────────────────────┘                 │
│                                                             │
│   1. Request arrives (founderId, personaId, message)        │
│   2. Load workspace from MongoDB                            │
│   3. Filter files for persona                               │
│   4. Assemble system prompt                                 │
│   5. Call OpenAI with prompt + history                      │
│   6. Stream response back                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Frontend (Next.js on Vercel)

```
/app
├── (auth)/                    # Clerk auth pages
├── (main)/
│   ├── onboarding/           # Discovery flow
│   ├── workspace/            # File viewer
│   ├── chat/                 # Persona chat UI
│   └── settings/             # Account settings
└── api/
    ├── onboarding/           # Generate workspace
    ├── workspace/            # CRUD files
    └── chat/                 # LLM interaction
```

### 2. Database (MongoDB Atlas)

```javascript
// workspaces collection
{
  _id: ObjectId,
  founderId: "user_123",           // Clerk user ID
  createdAt: ISODate,
  onboardingComplete: Boolean,
  
  // Workspace files as embedded documents
  files: {
    "FOUNDER.md": {
      content: "# Founder Profile\n...",
      updatedAt: ISODate
    },
    "COMPANY.md": {
      content: "# Company\n...",
      updatedAt: ISODate
    },
    "personas/grace/SOUL.md": {
      content: "# Grace\n...",
      updatedAt: ISODate
    }
    // ... more files
  }
}

// sessions collection (conversation history)
{
  _id: ObjectId,
  founderId: "user_123",
  personaId: "grace",
  messages: [
    { role: "user", content: "...", timestamp: ISODate },
    { role: "assistant", content: "...", timestamp: ISODate }
  ],
  createdAt: ISODate,
  updatedAt: ISODate
}
```

### 3. Router & Assembler

```typescript
// lib/workspace/router.ts
export async function routeRequest(req: ChatRequest) {
  const { founderId, personaId, message } = req;
  
  // 1. Load workspace (scoped by founderId)
  const workspace = await db.workspaces.findOne({ founderId });
  
  // 2. Get relevant files for this persona
  const files = filterFilesForPersona(workspace.files, personaId);
  
  // 3. Assemble system prompt
  const systemPrompt = assemblePrompt(files);
  
  // 4. Load conversation history
  const history = await getSessionHistory(founderId, personaId);
  
  // 5. Call LLM
  return callLLM(systemPrompt, history, message);
}
```

### 4. Prompt Assembly

```typescript
// lib/workspace/assembler.ts
export function assemblePrompt(files: WorkspaceFiles): string {
  const sections = [];
  
  // Persona identity first
  if (files['personas/{persona}/IDENTITY.md']) {
    sections.push(files['personas/{persona}/IDENTITY.md']);
  }
  
  // Persona soul (how they think)
  if (files['personas/{persona}/SOUL.md']) {
    sections.push(files['personas/{persona}/SOUL.md']);
  }
  
  // Founder context
  if (files['FOUNDER.md']) {
    sections.push('# About the Founder\n' + files['FOUNDER.md']);
  }
  
  // Company context
  if (files['COMPANY.md']) {
    sections.push('# Company Context\n' + files['COMPANY.md']);
  }
  
  // Shared memory
  if (files['CONTEXT.md']) {
    sections.push('# Recent Context\n' + files['CONTEXT.md']);
  }
  
  // Persona tools
  if (files['personas/{persona}/TOOLS.md']) {
    sections.push('# My Capabilities\n' + files['personas/{persona}/TOOLS.md']);
  }
  
  return sections.join('\n\n---\n\n');
}
```

## Scaling

### Why It Scales

1. **Stateless compute**: Vercel functions have no local state
2. **Shared database**: MongoDB Atlas handles all persistence
3. **API-based LLM**: OpenAI scales infinitely
4. **Horizontal scaling**: Add more Vercel instances = done

### Data Isolation

```typescript
// Every query scoped by founderId
const workspace = await db.workspaces.findOne({ 
  founderId: auth.userId  // Always required
});

// Middleware enforces this
export function withFounderScope(handler) {
  return async (req) => {
    const { userId } = auth();
    const founderId = userId;  // Or map to founderId
    
    // Inject scoped DB into request
    req.db = createScopedDb(founderId);
    return handler(req);
  };
}
```

## API Key Management

### Current: Platform Pays

```typescript
// All requests use our API key
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

// Track usage per founder
await db.usage.insertOne({
  founderId,
  tokens: response.usage.total_tokens,
  cost: calculateCost(response.usage)
});
```

### Future: BYOK (Bring Your Own Key)

```typescript
async function getLLMClient(founderId: string) {
  const founder = await getFounder(founderId);
  
  if (founder.integrations?.openai?.apiKey) {
    // Their key (encrypted at rest)
    return new OpenAI({
      apiKey: decrypt(founder.integrations.openai.apiKey)
    });
  }
  
  // Platform key (default)
  return new OpenAI({
    apiKey: process.env.OPENAI_API_KEY
  });
}
```

## Security

### Authentication
- Clerk handles all auth
- JWT tokens for API requests
- Webhook for user events

### Data Protection
- API keys encrypted at rest (AES-256)
- All queries scoped by founderId
- No cross-tenant data access possible
- HTTPS everywhere

### Isolation Levels

| Level | Implementation | When |
|-------|----------------|------|
| Query scoping | founderId on every query | Default |
| Middleware | Automatic injection | Default |
| Separate collections | Per-founder collections | Enterprise |
| Separate databases | Per-founder DB | Compliance |
