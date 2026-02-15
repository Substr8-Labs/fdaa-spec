# TowerHQ File Schema

## Overview

Each founder has a **workspace** containing markdown files that define:
- Who they are (FOUNDER.md)
- What they're building (COMPANY.md)
- How each persona thinks and works (personas/*)

## Workspace Structure

```
{founderId}/
├── FOUNDER.md              # Founder profile
├── COMPANY.md              # Company/product context
├── CONTEXT.md              # Shared memory across personas
├── STRATEGY.md             # Current priorities (optional)
│
└── personas/
    ├── ada/
    │   ├── IDENTITY.md     # Name, emoji, role
    │   ├── SOUL.md         # Personality, values, style
    │   ├── TOOLS.md        # Capabilities and integrations
    │   └── MEMORY.md       # Ada's personal notes
    │
    ├── grace/
    │   ├── IDENTITY.md
    │   ├── SOUL.md
    │   ├── TOOLS.md
    │   └── MEMORY.md
    │
    ├── tony/
    │   └── ...
    │
    └── val/
        └── ...
```

## File Specifications

### FOUNDER.md

**Purpose:** Who is this founder? What do they need?

**Template:**
```markdown
# Founder Profile

- **Name:** {founderName}
- **Timezone:** {timezone}
- **Stage:** {stage}

## What I'm Building
{whatBuilding}

## Who I'm Building For
{targetCustomer}

## Current Blocker
{biggestBlocker}

## Working Style
{workingStyle}
```

**Generated from:** Onboarding Q&A

---

### COMPANY.md

**Purpose:** Product context the personas reference

**Template:**
```markdown
# {companyName}

## What It Is
{whatBuilding}

## Target Customer (ICP)
{targetCustomer}

## Current Stage
{stage}

## Positioning
{positioning}

## Key Decisions
*(Logged as decisions are made)*

- {date}: {decision}
```

**Updated by:** Personas log decisions here

---

### CONTEXT.md

**Purpose:** Shared memory all personas can read

**Template:**
```markdown
# Shared Context

## Current Focus
{currentFocus}

## Recent Discussions
- {date}: {topic} - {outcome}

## Open Questions
- {question}
```

**Updated by:** Personas after significant conversations

---

### personas/{persona}/IDENTITY.md

**Purpose:** Who is this persona?

**Template:**
```markdown
# {personaName} — {role}

- **Name:** {personaName}
- **Emoji:** {emoji}
- **Role:** {role}
- **Reports to:** {founderName}

{shortDescription}
```

**Personas:**
| Persona | Role | Emoji |
|---------|------|-------|
| Ada | CTO / Technical Lead | ✦ |
| Grace | Head of Product | 🚀 |
| Tony | Head of Marketing | 📣 |
| Val | Head of Operations | 🛡️ |

---

### personas/{persona}/SOUL.md

**Purpose:** How does this persona think and work?

**Template:**
```markdown
# {personaName} — How I Think

## My Role
{roleDescription}

## My Values
- {value1}
- {value2}
- {value3}

## How I Work
- {workStyle1}
- {workStyle2}

## What I Won't Do
- {boundary1}
- {boundary2}
```

**Each persona has distinct values:**
- **Ada:** Ship over perfect, simplicity first, honest assessment
- **Grace:** User insight > intuition, small scope, fast ships
- **Tony:** Hook over explanation, authenticity, distribution focus
- **Val:** Systems over heroics, risk awareness, efficiency

---

### personas/{persona}/TOOLS.md

**Purpose:** What can this persona do?

**Template:**
```markdown
# {personaName}'s Capabilities

## What I Can Help With
- {capability1}
- {capability2}

## Integrations
- {integration1}: {description}
- {integration2}: {description}
```

**Persona capabilities:**
| Persona | Capabilities |
|---------|--------------|
| Ada | Architecture, code review, debugging, tech decisions |
| Grace | Product strategy, prioritization, user stories, positioning |
| Tony | Copywriting, social media, landing pages, launches |
| Val | Process docs, checklists, risk assessment, coordination |

---

### personas/{persona}/MEMORY.md

**Purpose:** Persona-specific notes and learnings

**Template:**
```markdown
# {personaName}'s Notes

## Things I've Learned About {founderName}
- {learning}

## Decisions I've Helped Make
- {date}: {decision}

## My Observations
- {observation}
```

**Updated by:** The persona after conversations

---

## File Injection Order

When assembling a system prompt:

```
1. personas/{persona}/IDENTITY.md   # Who am I?
2. personas/{persona}/SOUL.md       # How do I think?
3. FOUNDER.md                       # Who am I helping?
4. COMPANY.md                       # What are we building?
5. CONTEXT.md                       # Recent context
6. personas/{persona}/MEMORY.md     # My notes
7. personas/{persona}/TOOLS.md      # What can I do?
```

## MongoDB Storage

```javascript
{
  founderId: "user_123",
  files: {
    "FOUNDER.md": { content: "...", updatedAt: ISODate },
    "COMPANY.md": { content: "...", updatedAt: ISODate },
    "CONTEXT.md": { content: "...", updatedAt: ISODate },
    "personas/ada/IDENTITY.md": { content: "...", updatedAt: ISODate },
    "personas/ada/SOUL.md": { content: "...", updatedAt: ISODate },
    "personas/ada/TOOLS.md": { content: "...", updatedAt: ISODate },
    "personas/ada/MEMORY.md": { content: "...", updatedAt: ISODate },
    // ... more files
  }
}
```

## Template Engine

Files can be generated using templates (Jinja2 or TypeScript):

```typescript
// Static structure
const template = `
# Founder Profile

- **Name:** ${answers.founderName}
- **Stage:** ${answers.stage}

## What I'm Building
${answers.whatBuilding}
`;

// Or Jinja for more complex logic
const template = `
# {{ founder.name }}

{% if founder.blocker %}
## Current Blocker
{{ founder.blocker }}
{% endif %}
`;
```

## Evolution

Files evolve over time:
1. **Onboarding:** Initial files generated from Q&A
2. **Conversations:** Personas update MEMORY.md, CONTEXT.md
3. **Decisions:** Logged in COMPANY.md
4. **Founder edits:** Direct edits via workspace UI
