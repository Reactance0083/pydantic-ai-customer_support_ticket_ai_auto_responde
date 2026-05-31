# Customer Support Ticket → AI Auto-Responder with Human Handoff

A production-ready FastAPI + pydantic-ai template that ingests support tickets from Zendesk, Intercom, or Help Scout, generates grounded responses with RAG over your docs, scores confidence, and either auto-replies or escalates to a human with a pre-written draft.

## Overview

Most "AI customer support" tools either hallucinate confidently or escalate everything. This template ships a calibrated agent that:

- Retrieves relevant docs before answering (RAG, no freelancing)
- Scores its own confidence against a structured rubric
- Auto-sends replies above a configurable threshold
- Escalates low-confidence tickets to humans with a draft reply attached
- Logs every decision (ticket, retrieved chunks, confidence, action) for audit

Stack: **FastAPI** + **pydantic-ai** + **OpenAI** (swappable) + **ChromaDB** (local vector store) + **SQLite** (audit log).

## What it does

1. Receives a ticket via webhook (`/webhook/{provider}`) or direct POST (`/tickets`)
2. Normalizes provider-specific payloads into a canonical `Ticket` schema
3. Runs a pydantic-ai agent that:
   - Retrieves top-k doc chunks from the vector store
   - Generates a draft reply
   - Self-scores confidence on a 0.0–1.0 rubric (intent clarity, doc coverage, ambiguity, risk)
4. Branches:
   - `confidence >= AUTO_REPLY_THRESHOLD` → posts reply back to provider, closes ticket
   - Otherwise → assigns to human queue with draft attached
5. Persists ticket, retrieved context, confidence, and action to the audit log

## Prerequisites

- Python 3.11+
- An OpenAI API key (or Anthropic — see Customization)
- Optional: Zendesk / Intercom / Help Scout API credentials for sending replies back

## Setup

1. **Clone and enter the repo**
   ```bash
   git clone https://github.com/yourname/support-autoresponder.git
   cd support-autoresponder
   ```

2. **Create a virtualenv and install dependencies**
   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   ```

3. **Copy and fill in env variables**
   ```bash
   cp .env.example .env
   # edit .env — at minimum set OPENAI_API_KEY
   ```

4. **Drop your docs into `./docs/`** (Markdown, `.txt`, or `.html`)
   ```bash
   cp ~/path/to/your/help-center/*.md docs/
   ```

5. **Index your docs into the vector store**
   ```bash
   python -m app.ingest
   ```

6. **Run the server**
   ```bash
   uvicorn app.main:app --reload --port 8000
   ```

7. **Verify it's up**
   ```bash
   curl http://localhost:8000/health
   ```

You're live. To connect Zendesk/Intercom, point their webhook at `https://your-host/webhook/zendesk` (or `/intercom`, `/helpscout`).

## Usage

Send a test ticket:

```bash
curl -X POST http://localhost:8000/tickets \
  -H "Content-Type: application/json" \
  -d '{
    "ticket_id": "t_001",
    "subject": "Can't reset my password",
    "body": "I clicked forgot password 3 times, no email arrives. Using user@acme.com.",
    "customer_email": "user@acme.com",
    "channel": "email"
  }'
```

Response:

```json
{
  "ticket_id": "t_001",
  "action": "auto_reply",
  "confidence": 0.87,
  "reply": "Hi — password reset emails can take up to 5 minutes...",
  "sources": ["docs/password-reset.md", "docs/email-deliverability.md"]
}
```

Low-confidence tickets return `"action": "escalate"` with the draft reply for the agent to review.

## API Endpoints

### `GET /health`
Liveness check.
```bash
curl http://localhost:8000/health
```

### `POST /tickets`
Process a ticket directly (canonical schema). Useful for testing or custom integrations.
```bash
curl -X POST http://localhost:8000/tickets \
  -H "Content-Type: application/json" \
  -d '{"ticket_id":"t_42","subject":"Refund question","body":"How do I get a refund for last month?","customer_email":"a@b.com","channel":"email"}'
```

### `POST /webhook/{provider}`
Provider-specific webhook. `provider` ∈ `zendesk | intercom | helpscout`.
```bash
curl -X POST http://localhost:8000/webhook/zendesk \
  -H "Content-Type: application/json" \
  -H "X-Zendesk-Webhook-Signature: ..." \
  -d @sample_payloads/zendesk_ticket_created.json
```

### `GET /tickets/{ticket_id}`
Fetch the audit record for a processed ticket.
```bash
curl http://localhost:8000/tickets/t_001
```

### `GET /tickets?action=escalate&limit=20`
List recent tickets, filterable by action (`auto_reply` | `escalate`).
```bash
curl "http://localhost:8000/tickets?action=escalate&limit=20"
```

### `POST /reindex`
Re-ingest the `./docs/` directory after updates.
```bash
curl -X POST http://localhost:8000/reindex
```

Python (`httpx`) example:

```python
import httpx

resp = httpx.post("http://localhost:8000/tickets", json={
    "ticket_id": "t_99",
    "subject": "Billing issue",
    "body": "Charged twice for May.",
    "customer_email": "user@acme.com",
    "channel": "email",
})
print(resp.json())
```

## Configuration

All settings live in `.env`:

| Variable | Default | Description |
|---|---|---|
| `OPENAI_API_KEY` | — | Required. Used by pydantic-ai agent and embeddings. |
| `MODEL_NAME` | `openai:gpt-4o-mini` | pydantic-ai model identifier. |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | Embedding model for RAG. |
| `AUTO_REPLY_THRESHOLD` | `0.75` | Min confidence (0–1) to auto-send. Below → escalate. |
| `RAG_TOP_K` | `4` | Doc chunks retrieved per query. |
| `CHUNK_SIZE` | `800` | Token chunk size during ingestion. |
| `CHUNK_OVERLAP` | `100` | Token overlap between chunks. |
| `VECTOR_DB_PATH` | `./data/chroma` | Local ChromaDB directory. |
| `AUDIT_DB_URL` | `sqlite:///./data/audit.db` | SQLAlchemy URL for audit log. |
| `DOCS_DIR` | `./docs` | Source docs for RAG ingestion. |
| `ZENDESK_SUBDOMAIN` | — | Optional. Required to post replies back to Zendesk. |
| `ZENDESK_EMAIL` | — | Optional. Zendesk agent email. |
| `ZENDESK_API_TOKEN` | — | Optional. Zendesk API token. |
| `INTERCOM_ACCESS_TOKEN` | — | Optional. Intercom posting. |
| `HELPSCOUT_API_KEY` | — | Optional. Help Scout posting. |
| `WEBHOOK_SECRET` | — | Optional. HMAC verification for incoming webhooks. |
| `LOG_LEVEL` | `INFO` | Standard Python log level. |

## Customization

**Tune confidence rubric.** The agent's self-scoring is defined in `app/agent.py` as a structured Pydantic output (`intent_clarity`, `doc_coverage`, `ambiguity`, `risk`). Adjust the weights or add fields — the prompt instructs the model to fill them, and `compute_confidence()` aggregates.

**Change the threshold.** Set `AUTO_REPLY_THRESHOLD=0.85` to be more conservative. Tickets just below threshold still produce a draft for the human.

**Swap the LLM.** Set `MODEL_NAME=anthropic:claude-3-5-sonnet-latest` and add `ANTHROPIC_API_KEY`. pydantic-ai handles the rest.

**Swap the vector store.** Replace `app/rag.py`'s ChromaDB client with Pinecone, Qdrant, or pgvector — the interface is `retrieve(query, k) -> list[Chunk]`.

**Add a provider.** Implement `app/providers/<name>.py` with `parse_webhook(payload) -> Ticket` and `send_reply(ticket_id, reply) -> None`, then register in `app/providers/__init__.py`.

**Add tools to the agent.** pydantic-ai tools live in `app/agent.py`. Add e.g. `lookup_order(order_id)` to let the agent fetch live data before replying.

**Custom escalation routing.** Edit `app/handoff.py` to assign to specific queues, Slack channels, or PagerDuty based on tags, customer tier, or topic.

## License

MIT. Use it, fork it, sell what you build with it.