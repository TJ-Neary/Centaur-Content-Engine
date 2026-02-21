# Architecture

> System design documentation for Centaur — how the system is structured and why.

---

## Design Philosophy

Three principles guide every architectural decision:

1. **Human-in-the-Loop First** — Every piece of content requires human approval before publication. This is a deliberate constraint, not a limitation. Platform algorithms penalize detectable AI content, and authentic voice requires human judgment. The system produces drafts, not posts.

2. **Local-First with Cloud Fallback** — Local models handle research and first drafts (cost-free, private, low-latency). Cloud APIs handle quality-critical tasks where accuracy matters most. Tasks route based on requirements with graceful degradation between providers.

3. **Composition over Monolith** — The system is composed of specialized modules that communicate through defined interfaces. Each module does one thing well. New capabilities are added as new modules, not by expanding existing ones.

---

## System Context (C4 Level 1)

```mermaid
graph TB
    creator[Content Creator]
    centaur[Centaur<br/>Content Operations Platform]
    linkedin[LinkedIn]
    youtube[YouTube]
    social[Social Media Platforms]
    newsletter[Newsletter]
    analytics[Platform Analytics]
    sources[Research Sources<br/>Articles, PDFs, Courses]
    crm[CRM System]
    resume[Experience Database]

    creator -->|Reviews, edits, approves| centaur
    centaur -->|Drafts for review| creator
    centaur -->|Formatted content| linkedin
    centaur -->|Adapted content| youtube
    centaur -->|Adapted content| social
    centaur -->|Deep-dive content| newsletter
    analytics -->|Performance data| centaur
    sources -->|Research material| centaur
    centaur <-->|Contact intelligence| crm
    centaur <-->|Experience data| resume

    style centaur fill:#1a1a2e,stroke:#58A6FF,stroke-width:2px,color:#fff
    style creator fill:#0d1117,stroke:#A371F7,stroke-width:2px,color:#fff
```

**External integrations** connect via MCP (Model Context Protocol) for real-time data exchange. The CRM integration enables engagement-driven contact scoring. The experience database provides authentic source material for content creation.

---

## Container Architecture (C4 Level 2)

```mermaid
graph TB
    subgraph Centaur Platform
        cli[CLI Interface<br/>Typer + Rich]

        subgraph Content Engine
            generator[Post Generator]
            pipeline[Writer-Critic-Reviser<br/>Pipeline]
            enforcer[Brand Enforcer]
            templates[Content Templates]
            optimizer[Content Optimizer]
            calendar[Content Calendar]
        end

        subgraph Knowledge Layer
            style_sys[Style System<br/>Blueprint + Few-Shot]
            knowledge[Knowledge Library<br/>Learning Modules]
            inbox[Research Inbox]
        end

        subgraph Analytics Engine
            health[Strategy Health Engine]
            tracker[Engagement Tracker]
            store[Analytics Store]
        end

        subgraph Infrastructure
            llm[LLM Abstraction Layer<br/>Ollama / Claude / Gemini]
            security[Security Toolkit<br/>PII Detection, Scanning]
            browser[Browser Automation<br/>Playwright MCP]
        end
    end

    cli --> generator
    generator --> pipeline
    pipeline --> enforcer
    enforcer --> calendar

    style_sys --> generator
    style_sys --> pipeline
    knowledge --> generator

    health --> store
    tracker --> store

    generator --> llm
    pipeline --> llm
    browser --> knowledge

    style cli fill:#1a1a2e,stroke:#58A6FF,color:#fff
    style pipeline fill:#1a1a2e,stroke:#A371F7,color:#fff
    style llm fill:#1a1a2e,stroke:#58A6FF,color:#fff
    style health fill:#1a1a2e,stroke:#58A6FF,color:#fff
```

---

## Key Design Decisions

| Decision | Choice | Rationale | Alternatives Considered |
|----------|--------|-----------|------------------------|
| **Content approval model** | Human-in-the-loop required | Platform algorithms penalize AI content; authenticity is the competitive moat | Full automation with quality thresholds — rejected |
| **Platform priority** | LinkedIn-first, adapt via transmutation | Direct professional impact; other platforms receive adapted content | Multi-platform from day one — rejected (scope) |
| **Voice preservation** | Few-shot learning with style analysis | Up to 23.5x accuracy improvement over zero-shot; immediate iteration without retraining | Model fine-tuning — deferred to Phase 4 when volume demands |
| **LLM strategy** | Local-first with cloud fallback | Cost control + privacy for routine tasks; quality optimization for critical tasks | API-only (cost), local-only (quality gaps) |
| **Quality assurance** | Multi-agent pipeline + deterministic rules | Specialized agents outperform single-pass generation; rule engine catches objective violations | Single-prompt with detailed instructions — insufficient quality |
| **Knowledge management** | File-system-first, Markdown, git-tracked | Zero dependencies, version-controlled evolution, directly readable by all system components | Vector database (premature), Notion/Obsidian (external dependency) |
| **Analytics approach** | Platform export ingestion with anomaly detection | Richer data from native exports; longitudinal tracking with version attribution | Manual tracking (doesn't scale), API integration (restricted) |
| **Strategy evolution** | Data-driven with version tagging | Controlled comparison of strategy changes; decisions based on measured outcomes | Intuition-based iteration — rejected |
| **Browser automation** | Playwright via MCP | Real browser context for JS rendering, authentication, complex interactions | Custom scrapers (fragile), API integrations (limited availability) |
| **Security** | 9-phase pre-commit scanner | Defense-in-depth: secrets, PII, paths, dependencies, custom terms, compliance | Basic .gitignore rules — insufficient |

---

## Primary Data Flow

```mermaid
flowchart TD
    A[Research Sources] -->|Ingest| B[Knowledge Library]
    B -->|Topic Selection| C[Post Generator]
    C -->|Initial Draft| D[Writer Agent]
    D -->|Draft| E[Critic Agent]
    E -->|Score + Feedback| F{Quality Threshold?}
    F -->|Below| G[Reviser Agent]
    G -->|Revised Draft| E
    F -->|Met| H[Brand Enforcer]
    H -->|Validated| I[Content Queue]
    I -->|For Review| J[Human Review]
    J -->|Approved| K[Platform Distribution]
    K -->|Published| L[Analytics Collection]
    L -->|Performance Data| M[Strategy Health Engine]
    M -->|Insights| N[Strategy Updates]
    N -->|Inform| C

    style D fill:#1a1a2e,stroke:#A371F7,color:#fff
    style E fill:#1a1a2e,stroke:#58A6FF,color:#fff
    style G fill:#1a1a2e,stroke:#A371F7,color:#fff
    style H fill:#1a1a2e,stroke:#58A6FF,color:#fff
    style J fill:#0d1117,stroke:#A371F7,stroke-width:2px,color:#fff
    style M fill:#1a1a2e,stroke:#58A6FF,color:#fff
```

The pipeline forms a **closed feedback loop**: published content generates analytics data, which informs strategy adjustments, which shape future content generation. Every piece of content is tagged with the strategy and style versions that produced it, enabling controlled attribution analysis.

---

## LLM Abstraction

The system abstracts across three LLM providers through a unified interface:

```mermaid
graph LR
    app[Application Layer] --> provider[Provider<br/>Abstraction]
    provider --> ollama[Ollama<br/>Local / Apple Silicon]
    provider --> claude[Claude API<br/>Anthropic]
    provider --> gemini[Gemini API<br/>Google]

    style provider fill:#1a1a2e,stroke:#58A6FF,color:#fff
    style ollama fill:#0d1117,stroke:#58A6FF,color:#fff
    style claude fill:#0d1117,stroke:#A371F7,color:#fff
    style gemini fill:#0d1117,stroke:#4285F4,color:#fff
```

| Provider | Use Case | Advantage |
|----------|----------|-----------|
| **Ollama** | Research, first drafts, embeddings | Free, private, low-latency, runs on Apple Silicon |
| **Claude** | Style transfer, final polish | Highest accuracy for voice preservation |
| **Gemini** | Multimodal analysis, large context | Image processing, PDF analysis, 1M+ token window |

Tasks route based on requirements. If a preferred provider is unavailable, the system falls back gracefully to the next best option.

---

## Security Posture

Security is built into the development workflow, not bolted on after:

- **Pre-commit validation** — A 9-phase scanner runs before every commit, checking for secrets, PII, hardcoded paths, sensitive files, dangerous patterns, and compliance
- **Runtime protection** — Prompt injection detection, input sanitization, path traversal prevention, and secure file operations
- **PII detection** — Automated scanning using Presidio and spaCy to prevent personal data leaks
- **Dependency auditing** — Known vulnerability checks against requirements
- **Data isolation** — Content, strategy documents, and personal data are architecturally separated from code and never enter version control

---

## Cross-Project Integration

Centaur is designed to operate within a broader project ecosystem via MCP (Model Context Protocol):

```mermaid
graph TB
    centaur[Centaur<br/>Content Engine]
    crm[CRM System<br/>Contact Management]
    resume[Resume Engine<br/>Experience Database]
    assistant[Personal AI Assistant<br/>Daily Operations]

    centaur <-->|Engagement scoring<br/>Contact intelligence| crm
    centaur <-->|Authentic experience data<br/>Content attribution| resume
    centaur <-->|Content briefings<br/>Voice commands<br/>Meeting prep| assistant

    style centaur fill:#1a1a2e,stroke:#58A6FF,stroke-width:2px,color:#fff
    style crm fill:#0d1117,stroke:#58A6FF,color:#fff
    style resume fill:#0d1117,stroke:#A371F7,color:#fff
    style assistant fill:#0d1117,stroke:#58A6FF,color:#fff
```

Integration workflows are designed but privacy-preserving — each system shares only the minimum data needed for its function. The content engine never receives raw CRM data; it receives engagement context. The resume engine provides experience summaries, not full records.

---

*Architecture documentation reflects the current system design. Source code is maintained in a private repository.*

Copyright 2026 TJ Neary. All Rights Reserved.
