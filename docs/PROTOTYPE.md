# TowerHQ Prototype Spec

## Prototype v1 (Completed)

**Status:** ✅ Deployed

**Stack:**
- Next.js on Vercel
- Postgres (Neon)
- Clerk auth
- OpenAI API

**Features:**
- [x] Onboarding flow (6 questions)
- [x] Workspace file generation
- [x] Workspace file viewer
- [x] Persona chat with context injection
- [x] Basic prompt assembly

**URLs:**
- Production: https://towerhq.substr8labs.com
- Onboarding: /onboarding
- Workspace: /workspace

---

## Prototype v2 (Next)

**Goal:** Multi-tenant MongoDB architecture

### Phase 1: MongoDB Migration

**Tasks:**

1. **Set up MongoDB Atlas**
   ```
   - Create free M0 cluster
   - Get connection string
   - Add to Vercel env vars
   ```

2. **Create MongoDB schema**
   ```javascript
   // workspaces collection
   {
     founderId: String,
     onboardingComplete: Boolean,
     files: {
       "path": { content: String, updatedAt: Date }
     },
     createdAt: Date,
     updatedAt: Date
   }
   
   // sessions collection
   {
     founderId: String,
     personaId: String,
     messages: [{ role, content, timestamp }]
   }
   ```

3. **Update API routes**
   - `/api/onboarding` → Write to MongoDB
   - `/api/workspace` → Read/write MongoDB
   - `/api/chat` → Load from MongoDB

4. **Update prompt assembler**
   ```typescript
   // Before: Postgres
   const files = await prisma.workspaceFile.findMany({ where: { workspaceId }});
   
   // After: MongoDB
   const workspace = await db.workspaces.findOne({ founderId });
   const files = workspace.files;
   ```

### Phase 2: Router & Isolation

**Tasks:**

1. **Create request router**
   ```typescript
   export async function routeChat(req: ChatRequest) {
     const { founderId, personaId, message } = req;
     
     // Load scoped workspace
     const workspace = await getWorkspace(founderId);
     
     // Filter for persona
     const files = filterForPersona(workspace.files, personaId);
     
     // Assemble and call
     const prompt = assemblePrompt(files);
     return callLLM(prompt, history, message);
   }
   ```

2. **Add scoping middleware**
   ```typescript
   export function withFounderScope(handler) {
     return async (req) => {
       const founderId = auth().userId;
       req.scope = { founderId };
       return handler(req);
     };
   }
   ```

3. **Test isolation**
   - Create two test founders
   - Verify they can't see each other's data
   - Verify each persona loads correct files

### Phase 3: Session Management

**Tasks:**

1. **Store conversation history**
   ```javascript
   // sessions collection
   {
     _id: ObjectId,
     founderId: "user_123",
     personaId: "grace",
     messages: [
       { role: "user", content: "...", timestamp: Date },
       { role: "assistant", content: "...", timestamp: Date }
     ]
   }
   ```

2. **Load history in chat**
   ```typescript
   const session = await db.sessions.findOne({ founderId, personaId });
   const history = session?.messages || [];
   ```

3. **Truncate history for context window**
   ```typescript
   const MAX_HISTORY = 20;
   const recentHistory = history.slice(-MAX_HISTORY);
   ```

---

## API Specification

### POST /api/onboarding

**Request:**
```json
{
  "answers": {
    "founderName": "Sarah",
    "companyName": "LegalScribe",
    "whatBuilding": "AI writing assistant for legal docs",
    "targetCustomer": "Solo attorneys",
    "stage": "launched",
    "biggestBlocker": "Pricing strategy"
  }
}
```

**Response:**
```json
{
  "success": true,
  "workspaceId": "ws_abc123",
  "filesCreated": 15,
  "message": "Workspace created! Your C-Suite is ready."
}
```

### GET /api/workspace

**Response:**
```json
{
  "founderId": "user_123",
  "onboardingComplete": true,
  "files": [
    { "path": "FOUNDER.md", "content": "...", "updatedAt": "..." },
    { "path": "personas/grace/SOUL.md", "content": "...", "updatedAt": "..." }
  ]
}
```

### PATCH /api/workspace

**Request:**
```json
{
  "path": "CONTEXT.md",
  "content": "# Updated context\n..."
}
```

### POST /api/chat

**Request:**
```json
{
  "personaId": "grace",
  "message": "Help me figure out pricing"
}
```

**Response:** Server-sent events stream

---

## Environment Variables

```env
# MongoDB
MONGODB_URI=mongodb+srv://...

# Auth
CLERK_SECRET_KEY=sk_...
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_...

# LLM
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

---

## Testing Plan

### Unit Tests
- [ ] Prompt assembler
- [ ] File template generation
- [ ] Query scoping

### Integration Tests
- [ ] Onboarding flow
- [ ] Workspace CRUD
- [ ] Chat with history

### Manual Testing
- [ ] Two founders, verify isolation
- [ ] All four personas
- [ ] File editing reflects in chat

---

## Deployment

1. Push to GitHub
2. Vercel auto-deploys
3. Verify MongoDB connection
4. Test with real users

---

## Success Criteria

Prototype v2 is successful when:
1. ✅ Founders can complete onboarding
2. ✅ Workspace files are generated
3. ✅ Personas know founder context
4. ✅ Data is isolated between founders
5. ✅ Conversations persist across sessions
