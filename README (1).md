# Internal Knowledge Assistant — Petalnex RAG System

**AI Automation Internship — Petalnex Pvt. Ltd.**
**Author:** Abdul Wadood

An AI-powered internal knowledge assistant that lets employees ask natural-language questions about HR policy, internal procedures, and company information, and get accurate, source-cited answers in seconds — instead of digging through scattered documents or interrupting HR and senior staff.

---

## 1. Business Problem

A large chunk of every employee's week disappears into hunting for internal information — HR rules, technical procedures, you name it — spread across too many disconnected systems. This fragmented setup drains productivity, wears down employee patience, and keeps pulling senior staff and HR away from their real work to answer the same questions over and over.

**Key pain points:**
- **Information Silos & Search Friction** — basic keyword search breaks down when phrasing doesn't match exactly ("time off" vs "Leave Absence Policy").
- **The "Shoulder-Tap" Productivity Drain** — senior staff and HR get interrupted constantly with repetitive questions, breaking deep focus.
- **Outdated Information Reliance** — since finding the right document is hard, employees default to memory or asking a coworker, often following outdated policy.
- **The Hallucination Risk** — a plain LLM isn't safe for internal questions, since it will confidently invent policies that don't exist.

## 2. Existing Process

Employees currently search manually across Google Drive folders, intranet pages, and PDF manuals. When keyword search fails, they message a manager or HR on Slack/email and wait for a reply — often 15+ minutes for something that should take seconds.

## 3. Proposed Solution

A Retrieval-Augmented Generation (RAG) pipeline that:
1. Ingests company policies/SOPs into a vector database (Supabase pgvector).
2. On each employee question, retrieves only the relevant passages via similarity search.
3. Feeds that verified context — and only that context — to an LLM to generate a grounded answer.
4. Cites the source category for every answer, so it can be trusted and verified.
5. Falls back to an explicit "I don't know" response (rather than guessing) when no relevant match is found, and escalates unanswered questions to a human.

## 4. Architecture

```
React User Interface
        │
        ▼
   n8n Webhook  ──(auth header check)──► Reject Unauthorized (401)
        │
        ▼
  Validate Query Input (Filter)
        │
        ▼
  Supabase Vector Search  ◄────────────► Supabase pgvector Database
        │
        ▼
  AI Context Injector (Groq LLM + grounded prompt)
        │
        ▼
  Webhook Response ──► React User Interface

  If No Match:
  Vector Search ──► Support Ticket / HR Email ──► LLM API (escalation notice)
```

See `/docs/architecture-diagram.png` for the full visual diagram.

## 5. Technologies Used

| Category | Technology | Purpose |
|---|---|---|
| Frontend | React | Accepts employee questions, renders AI responses |
| Orchestration | n8n (self-hosted, Docker) | Webhook routing, validation, AI chaining, database queries |
| Vector Database | Supabase (pgvector) | Stores embedded company documents for similarity search |
| LLM | Groq (`openai/gpt-oss-120b`) | Generates grounded natural-language answers |
| Embeddings | Google Gemini (`gemini-embedding-001`) | Converts text into vector representations |
| Notifications | Gmail | HR escalation alerts for unanswered queries |

## 6. Workflows

This repository includes the following exported n8n workflows (`/workflows`):

| File | Purpose |
|---|---|
| `document-ingestion.json` | Converts company knowledge entries into embeddings and stores them in Supabase |
| `knowledge-assistant-core.json` | Webhook intake, validation, authentication |
| `knowledge-assistant-ai.json` | RAG retrieval + grounded AI answer generation (feature-complete pipeline) |
| `error-handler.json` | Global error-trigger workflow for logging and admin alerts |

## 7. AI Integration

- **Retrieval-Augmented Generation (RAG):** the LLM never answers from open-ended knowledge — it is restricted to the context retrieved from the vector database for that specific question.
- **Grounding rules:** the system prompt explicitly instructs the model to answer *only* from provided context and to state clearly when information isn't available, rather than inventing an answer.
- **Source citation:** every answer references the knowledge category (e.g. HR Policy, Products & Services, Support & Billing) it was drawn from.
- **Human escalation:** questions with no relevant match are logged and routed to HR instead of receiving a guessed answer.

## 8. Error Handling

- **Retry on Fail** is enabled on external API calls (LLM, vector search) — up to 3 attempts before failing.
- **Continue on Fail** routes failed nodes to a dedicated error branch instead of crashing the whole execution.
- A **dedicated Error Trigger workflow** (`error-handler.json`) catches any unhandled failure across linked workflows and sends an admin notification email.
- Full test coverage of failure scenarios (external API failure, database failure, invalid/missing data) is documented in `/docs/Test-Cases.pdf`.

## 9. Security

- All credentials (Supabase, Groq, Gemini, Gmail) are stored exclusively in n8n's encrypted credential manager — never hard-coded in workflow parameters or exposed in documentation/screenshots.
- The intake webhook requires a valid `x-api-key` header; requests without it are rejected with a 401-style response before any business logic runs.
- The Supabase connection used by this workflow is scoped to only what's needed for the knowledge base table.
- A full security audit (`/docs/Security-Audit.pdf`) was conducted, including rotation of a previously exposed API key discovered during development.

## 10. Business Benefits

- **Speed:** cuts policy lookup time from 15+ minutes to under 5 seconds.
- **Deflection:** reduces routine informational tickets/messages to HR and management.
- **Consistency:** employees always get the current, correct policy — not outdated information from memory or old documents.
- **Trust:** every answer is source-cited and grounded, with a transparent fallback when information isn't available.

## 11. Limitations

- Knowledge base currently covers a limited set of HR, product, and support entries (20+) — not yet the full document library.
- Strict JSON output enforcement (Structured Output Parser) is not yet applied to the live Q&A chain; answers currently return as formatted natural-language text with an inline citation rather than a strict schema.
- Human-escalation branch (auto ticket creation + HR email for unanswered questions) is designed but not fully wired into the production pipeline yet.
- Single-language support only.

## 12. Future Improvements

- Enforce strict structured JSON output end-to-end for easier downstream parsing.
- Complete the human-escalation branch with automatic ticket creation.
- Expand the knowledge base to the full document library via automated ingestion (e.g. scheduled sync from Google Drive).
- Add conversation memory so employees can ask natural follow-up questions.
- Move from Docker/local n8n to a production-grade hosted deployment with monitoring.

---

## Repository Structure

```
/
├── README.md
├── workflows/
│   ├── document-ingestion.json
│   ├── knowledge-assistant-core.json
│   ├── knowledge-assistant-ai.json
│   └── error-handler.json
├── docs/
│   ├── Project-Requirement-Document.pdf
│   ├── PRD-Architecture-Components.pdf
│   ├── architecture-diagram.png
│   ├── Test-Cases.pdf
│   └── Security-Audit.pdf
└── screenshots/
    └── (demo screenshots)
```
