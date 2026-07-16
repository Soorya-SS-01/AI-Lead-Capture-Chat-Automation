# LeadPilot AI — Instagram RAG Chat Agent & Lead Capture Automation

> An AI-powered Instagram DM automation system that answers customer questions using company-specific knowledge (RAG), automatically extracts and stores lead contact details, and syncs qualified leads to a CRM — all orchestrated with n8n.

*(Suggested project name — swap in whatever you're branding this as, e.g. "Firstday AI Instagram Agent". "LeadPilot AI" is just a placeholder that's short, catchy, and describes what it does.)*

---

## 📌 Overview

**LeadPilot AI** automates the first line of customer conversation on Instagram DMs for a business. Instead of a human manually replying to every "what's the price of X" message, the system:

1. Listens for incoming Instagram DMs via a webhook.
2. Answers customer questions **using the company's own product/knowledge data** (via Retrieval-Augmented Generation), instead of a generic or hallucinated AI answer.
3. Extracts a lead's **Name, Email, and Phone** from natural conversation and saves it to Google Sheets.
4. On a schedule, syncs those captured leads into a CRM (Zoho Bigin) so the sales team can follow up.

**Tech Stack:** n8n · Python/JavaScript · RAG (Qdrant) · Google Gemini · Google's `text-embedding-004` · Google Sheets · Zoho Bigin CRM

---

## 🖼️ Project Screenshots & Workflow Diagrams

> Drop your exported images into an `/assets` folder in the repo and update the paths below.

| | |
|---|---|
| **1. Main Workflow** — Instagram automation, RAG auto-reply, and lead capture in one n8n canvas | `![Main Workflow](assets/workflow-1-main.png)` |
| **2. Ingestion Workflow** — company data → embeddings → Qdrant | `![Ingestion Workflow](assets/workflow-2-ingestion.png)` |
| **3. CRM Sync Workflow** — scheduled Google Sheets → Zoho Bigin sync | `![CRM Sync Workflow](assets/workflow-3-crm-sync.png)` |
| **4. Live Demo** — real Instagram chat showing the bot responding | `![Instagram Chat Demo](assets/instagram-chat-demo.png)` |

---

## 🧠 Phase 1 — What is RAG, and Why This Project Needed It

**RAG (Retrieval-Augmented Generation)** is a technique that lets an AI model answer questions using *your own data*, instead of relying only on what it learned during training.

**Without RAG:** Ask an LLM "What's the price of the Samsung S25?" and it either doesn't know, or it guesses — a hallucination.

**With RAG:** The question is converted into an embedding, a vector database searches for the most relevant company document, that document is handed to the LLM as context, and the LLM answers *only* from that data — giving an accurate, company-specific reply.

### How RAG works in this project

| Step | What happens |
|---|---|
| 1 | Customer sends: *"What is the price of Samsung S25?"* |
| 2 | The message is converted into an embedding (a numerical vector) |
| 3 | That embedding is searched against Qdrant |
| 4 | Qdrant returns the closest matching document — e.g. *Samsung S25 — ₹74,999, free Buds worth ₹5,999* |
| 5 | That retrieved text + the original question is sent to Gemini |
| 6 | Gemini replies naturally: *"The Samsung S25 is ₹74,999 and currently includes free Galaxy Buds worth ₹5,999."* |

### Why Qdrant, and what "semantic search" means

Qdrant is a **vector database** — it stores embeddings instead of plain rows/columns, and retrieves by *meaning* rather than exact keyword match. A keyword search for "photography" fails to find a document that only mentions "200MP camera." A semantic search succeeds, because the embeddings for "photography" and "camera" land close together in vector space.

- `Photography ≈ Camera`
- `Cost ≈ Price`
- `Buy ≈ Purchase`

**RAG is the concept; Qdrant is one tool used to implement it.** Other implementation options considered: Chroma (local/small projects), Elasticsearch/OpenSearch (hybrid keyword + vector), FAISS (local vector index, no DB needed), or PostgreSQL + `pgvector` (SQL-based, good for smaller projects). Qdrant was chosen here for its fast semantic search and easy setup.

**Embeddings** are produced by a pretrained embedding model, not written by hand — this project uses **Google's `text-embedding-004`**. Both the company documents (at ingestion time) and every incoming customer question (at query time) are converted through the *same* embedding model, which is what makes the comparison meaningful.

---

## ⚙️ Phase 2 — How the Automation Actually Works

There are **three n8n workflows** working together.

### Workflow 1 — Main Automation (Instagram Webhook → RAG Reply + Lead Capture)

This is the core workflow. A single Instagram webhook event triggers two things in parallel: a **RAG-powered reply**, and a **lead-capture check**.

**0. Webhook verification (one-time setup by Meta)**
- `Instagram Webhook` receives Meta's verification handshake.
- `If` checks `hub.mode == subscribe` and `hub.verify_token` matches the configured secret.
- `Respond to Webhook` confirms the subscription back to Meta.

**Branch A — Answering the customer (RAG pipeline)**
1. `Get row(s) in sheet2` looks up the sender's Instagram ID in the **INSTAGRAM LEADS** Google Sheet.
2. `Is User Known` checks whether a matching row already exists.
   - **Unknown sender** → `Send Welcome Message` posts a greeting via the Instagram Graph API asking for Name, Phone, and Email.
   - **Known sender** → continues into the RAG flow:
     - `Embed User Question` sends the message text to Google's `text-embedding-004` model to get its embedding.
     - `Search Qdrant` searches the `firstday-rag` Qdrant collection for the closest matching document (top 1 result, with payload).
     - `AI Agent` (Gemini) receives the original question + the retrieved knowledge, and is instructed to answer **only** using that retrieved content, kept short (system prompt caps replies to roughly 100 characters — good for chat UX).
     - `HTTP Request1` posts Gemini's reply back to the customer via the Instagram Graph API `/messages` endpoint.

**Branch B — Capturing lead details**
1. `AI Agent1` (Gemini) acts as a **data-extraction engine**: it reads the raw incoming message and extracts `name`, `email`, and `phone` as strict JSON (nulls for anything not found).
2. `Code in JavaScript` strips any stray Markdown fences from the AI's output and safely parses it into a real JSON object — falling back to all-nulls if parsing fails, so the workflow never crashes on a bad AI response.
3. `If1` checks whether *at least one* of name/email/phone was actually found. If nothing was extracted, the branch stops here.
4. `Get row(s) in sheet` looks up the sender's Instagram ID in the same lead sheet.
5. `If2` checks whether that lookup found an existing row.
   - **No existing row** → `Append row in sheet` creates a new lead: Name, Phone, Email, InstagramID, LastUpdated.
   - **Existing row found** → `Update row in sheet` updates it, preferring the newly-extracted value but falling back to the previously stored value if the new extraction returned null (so a later message doesn't accidentally erase a previously captured phone number, for example).

### Workflow 2 — Knowledge Ingestion (Data → Embeddings → Qdrant)

This is the pipeline that *populates* the knowledge base Branch A searches against. Company data (product info, prices, offers, FAQs) is:
1. Read/prepared as text chunks.
2. Sent to the embedding model (Google `text-embedding-004`, or optionally OpenAI's `text-embedding-3-small`) to generate a vector for each chunk.
3. Upserted into the `firstday-rag` Qdrant collection along with the original text as payload, so Search Qdrant can retrieve it later.

*(Add your actual node-by-node breakdown here once you export this workflow's JSON — happy to document it the same way once you share it.)*

### Workflow 3 — Scheduled CRM Sync (Google Sheets → Zoho Bigin)

Runs on a **schedule trigger** (e.g. every few hours). It reads new or updated rows from the **INSTAGRAM LEADS** Google Sheet and pushes them into **Zoho Bigin CRM** (or any CRM), so the sales/marketing team works out of the CRM rather than a spreadsheet.

*(Same note — document the exact node flow here once exported.)*

---

## 🚧 Challenges & How They Were Solved

**"How does the AI know our company's information?"**
Early on, the instinct was to stuff the entire company knowledge base into every prompt sent to the AI. That doesn't scale — it burns through token/context limits fast and isn't a real solution. Researching alternatives led to RAG: instead of sending everything, only the *most relevant* piece of information is retrieved and sent, based on the customer's specific question. Several RAG implementation options were evaluated (vector DB, local vector index, hybrid search, SQL + embeddings) before settling on Qdrant for its simplicity and speed.

**"How do you know an Instagram conversation has ended?"**
Instagram DMs don't have a fixed "conversation over" event. The chosen approach is an **inactivity timeout**: if no new message arrives from the customer within a defined window (e.g. 10–15 minutes), the conversation is treated as complete. An alternative — detecting explicit closing phrases like "thank you" — was considered but judged less reliable than the timeout approach.

**"Is Google Sheets good enough for production?"**
No — it doesn't scale well and isn't built for concurrent, high-volume writes. It's used here as a fast, visual way to prototype and demo; the plan is to migrate to a proper database (see below).

---

## 🚀 Future Enhancements

- **Lead scoring** — classify each captured lead as High / Moderate / Low interest based on conversation sentiment, buying intent, and engagement, so the sales team can prioritize follow-ups.
- **Low-interest lead retention** — rather than discarding non-qualified conversations, log them separately so marketing can re-engage later if resources allow.
- **Move off Google Sheets** — migrate lead storage to a production-grade database such as **PostgreSQL**, **MySQL**, or **MongoDB**.
- **True sentiment analysis pass** — add an AI-based sentiment/buying-intent classification step on completed conversations, in addition to the current contact-detail extraction.

---

## 🔐 Setup Notes (Read Before Publishing This Repo)

- The exported n8n workflow JSON in this project currently has an API key and an Instagram access token **hardcoded directly in node parameters**. Before committing this repo publicly:
  - Rotate the Gemini API key and the Instagram Graph API token.
  - Replace hardcoded values with n8n **Credentials** or environment variables.
- Required credentials/services to run this yourself:
  - Instagram/Meta Graph API access token + verified webhook
  - Google Gemini API key
  - Qdrant instance (self-hosted or managed) + a collection for your knowledge base
  - Google Sheets API access (service account or OAuth)
  - Zoho Bigin (or your CRM of choice) API credentials

---

## 🧩 Quick Reference: Key Concepts

| Term | Meaning |
|---|---|
| **RAG** | Retrieving relevant company data first, then having the LLM generate an answer grounded in that data |
| **Embedding** | A numerical vector representation of text that captures meaning, produced by an embedding model |
| **Vector Database (Qdrant)** | A database built to store embeddings and perform semantic similarity search (via HNSW indexing + cosine/dot-product similarity) |
| **Semantic Search** | Finding results based on meaning, not exact keyword matches |
| **Prompt Engineering** | Structuring instructions/context given to an AI model to get consistent, accurate output |
