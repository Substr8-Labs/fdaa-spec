# TowerHQ Onboarding Flow

## Philosophy

> **The onboarding IS the first value.**
> 
> Founders should feel the "wow" before setup is complete.
> Not "configure your team" — feel the leverage.

## The Flow

```
┌─────────────────────────────────────────────────────────┐
│                    Onboarding Flow                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Step 1: The Hook                                       │
│  "What's been stuck for more than a week?"              │
│  → Captures the blocker (immediate relevance)           │
│                                                         │
│  Step 2: Identity                                       │
│  "What should we call you?"                             │
│  → Captures founder name                                │
│                                                         │
│  Step 3: Product                                        │
│  "What's the name of what you're building?"             │
│  "In one or two sentences, what is it?"                 │
│  → Captures company name + description                  │
│                                                         │
│  Step 4: Customer                                       │
│  "Who's your ideal customer?"                           │
│  → Captures ICP                                         │
│                                                         │
│  Step 5: Stage                                          │
│  "Where are you at?"                                    │
│  💡 Idea | 🔨 Building | 🚀 Launched | 📈 Scaling       │
│  → Captures lifecycle stage                             │
│                                                         │
│  Step 6: Generate                                       │
│  → System creates workspace files                       │
│  → Team introduces themselves                           │
│  → First persona addresses the blocker                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Question Design

### Why "What's stuck?" First

Traditional onboarding: "Tell us about yourself"
→ Generic, boring, feels like a form

TowerHQ onboarding: "What's stuck?"
→ Immediate relevance, emotional hook, sets up first value

### Question Sequence

| # | Question | Why |
|---|----------|-----|
| 1 | What's stuck? | Emotional hook, defines first task |
| 2 | Your name? | Personal, easy, builds rapport |
| 3 | What are you building? | Product context |
| 4 | Who's it for? | Customer context |
| 5 | What stage? | Lifecycle context |
| 6 | (Generate) | System takes over |

## UI Design

### Screen 1: The Hook

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   What's the one thing that's been stuck        │
│   for more than a week?                         │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │                                           │ │
│   │ [Large text input]                        │ │
│   │                                           │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│                            [Continue →]         │
│                                                 │
│   ───────────────────────────────────────────   │
│   Your AI C-Suite: Ada ✦ Grace 🚀 Tony 📣 Val 🛡️ │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Screen 5: Stage Selection

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   Where are you at?                             │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ 💡 Idea stage                             │ │
│   │    Still exploring and validating         │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ 🔨 Building                               │ │
│   │    Actively developing                    │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ 🚀 Launched                               │ │
│   │    Live with early users                  │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ 📈 Scaling                                │ │
│   │    Growing and optimizing                 │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
└─────────────────────────────────────────────────┘
```

## File Generation

After step 5, system generates:

```
FOUNDER.md              ← From name, working style
COMPANY.md              ← From product, ICP, stage
CONTEXT.md              ← From blocker (current focus)

personas/ada/
  IDENTITY.md           ← Standard template
  SOUL.md               ← Standard template
  TOOLS.md              ← Standard template
  MEMORY.md             ← Empty

personas/grace/
  ...

personas/tony/
  ...

personas/val/
  ...
```

## The Handoff

After files are generated, the team introduces themselves:

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   ✨ Your C-Suite is ready                      │
│                                                 │
│   Based on what you shared, here's your team:   │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ ✦ Ada (CTO)                               │ │
│   │ "I'll handle the technical architecture." │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ 🚀 Grace (Product)                        │ │
│   │ "That pricing problem? Already on it."    │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ 📣 Tony (Marketing)                       │ │
│   │ "When you're ready, I'll get you noticed."│ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│   ┌───────────────────────────────────────────┐ │
│   │ 🛡️ Val (Operations)                       │ │
│   │ "I'll make sure nothing falls through."   │ │
│   └───────────────────────────────────────────┘ │
│                                                 │
│                  [Talk to Grace about pricing →]│
│                                                 │
└─────────────────────────────────────────────────┘
```

## First Value

The CTA routes to the persona most relevant to their blocker:

| Blocker Type | Lead Persona | First Action |
|--------------|--------------|--------------|
| Pricing | Grace | Pricing framework |
| Positioning | Grace | Positioning statement |
| Technical | Ada | Architecture review |
| Marketing | Tony | Distribution channels |
| Overwhelmed | Val | Weekly sprint plan |

The first conversation already has context:

```
Grace: "I see pricing is what's been stuck. Let me help.

Based on what you told me about LegalScribe and solo attorneys,
here's a first take on pricing structure:

[Actual pricing framework]

What feels off about this?"
```

**That's the wow moment.** No setup, no explaining — the AI already knows.
