# APIHUB — Network API Bible
> Home AI Infrastructure · Fleet-wide Service Registry

**Machines:**
- **Node1** · <NODE1_IP> · Agents: <AGENT1>, <AGENT2>
- **Node2** · <NODE2_IP> · Agents: <AGENT3>, <AGENT4>, <AGENT5>

---

## Authentication
| Service        | Header      | Env Var            | Required | Machine |
|----------------|-------------|--------------------|----------|---------|
| agent-central  | `X-API-Key` | `SERVER_API_KEY`   | optional | Node1   |
| <AGENT1>           | `X-API-Key` | `AGENT1_API_KEY`     | optional | Node1   |
| <AGENT2>          | `X-API-Key` | `AGENT2_API_KEY`    | optional | Node1   |
| life-timeline  | `X-API-Key` | `TIMELINE_API_KEY` | required | Node1   |
| agent-central  | `X-API-Key` | `SERVER_API_KEY`   | optional | Node2   |
| <AGENT3>           | `X-API-Key` | `AGENT3_API_KEY`     | optional | Node2   |
| <AGENT4>   | `X-API-Key` | `AGENT4_API_KEY` | optional | Node2 |
| <AGENT5>         | `X-API-Key` | `AGENT5_API_KEY`   | optional | Node2   |

All agent-central APIs are Flask (HTTP/1.1). Life-timeline is FastAPI.
Rate limit on life-timeline: 60 req/min (GET), 30 req/min (POST/PUT/DELETE).

---

## agent-central — Server Status API
**Port:** `5000`  **Base:** `http://<NODE1_IP>:5000`

| Method | Endpoint  | Description                              |
|--------|-----------|------------------------------------------|
| GET    | `/status` | Server state, agent threads, queue depth |
| GET    | `/queue`  | Current dispatcher queue size            |

`/status` response:
```json
{
  "server": "agent-central",
  "cooldown_sec": 30,
  "queue_pending": 0,
  "agents": {"<AGENT1>": {"alive": true}, "<AGENT2>": {"alive": true}},
  "stopping": false
}
```

---

## <AGENT1> — Telegram Chat Agent API
**Port:** `5001`  **Base:** `http://<NODE1_IP>:5001`  **Model:** `qwen2.5:1.5b`  **Priority:** 1

| Method | Endpoint   | Description                              |
|--------|------------|------------------------------------------|
| GET    | `/status`  | Agent state, model, history length, queue |
| POST   | `/message` | Submit text message → queued, non-blocking |
| POST   | `/clear`   | Clear conversation history               |

`/message` body: `{"text": "your message"}`
`/status` response:
```json
{
  "agent": "<AGENT1>",
  "model": "qwen2.5:1.5b",
  "history": 12,
  "queue_size": 0,
  "running": true
}
```
Replies are sent to the configured Telegram chat (`AGENT1_CHAT_ID`).

---

## <AGENT2> — Email Campaign Agent API
**Port:** `5002`  **Base:** `http://<NODE1_IP>:5002`  **Model:** `qwen2.5:3b`  **Priority:** 0 (highest)

| Method | Endpoint   | Description                                    |
|--------|------------|------------------------------------------------|
| GET    | `/status`  | Agent state, model, Gmail status, queue        |
| POST   | `/message` | Submit text message → queued, non-blocking     |
| POST   | `/task`    | Submit structured email campaign task          |
| POST   | `/clear`   | Clear conversation history                     |

`/message` body: `{"text": "your message"}`
`/task` body:
```json
{
  "brief":    "Send intro email to leads",
  "template": "lead_outreach",
  "leads":    [{"first_name": "John", "email": "john@example.com"}]
}
```
`/status` response:
```json
{
  "agent": "<AGENT2>",
  "model": "qwen2.5:3b",
  "gmail": "not configured",
  "queue_size": 0,
  "running": true
}
```
Replies sent to Telegram chat (`AGENT2_CHAT_ID`). Email drafts sent for Telegram approval before sending.

---

## life-timeline — Personal Timeline API
**Port:** `8000`  **Base:** `http://<NODE1_IP>:8000`  **Auth:** required (`TIMELINE_API_KEY`)
**Framework:** FastAPI · **DB:** SQLite (SQLModel)

### Timeline
| Method | Endpoint                  | Description                    |
|--------|---------------------------|--------------------------------|
| GET    | `/`                       | Web UI (HTML, no auth)         |
| GET    | `/api/timeline`           | Full timeline + milestones + notes |
| GET    | `/api/timeline/current`   | Timeline metadata only         |
| PUT    | `/api/timeline`           | Update timeline name/dates     |

### Milestones
| Method | Endpoint                       | Description              |
|--------|--------------------------------|--------------------------|
| POST   | `/api/milestones`              | Create milestone         |
| PUT    | `/api/milestones/{id}`         | Update milestone         |
| DELETE | `/api/milestones/{id}`         | Delete milestone         |

### Notes
| Method | Endpoint              | Description   |
|--------|-----------------------|---------------|
| POST   | `/api/notes`          | Create note   |
| DELETE | `/api/notes/{id}`     | Delete note   |

`/api/timeline` response shape:
```json
{
  "timeline":   { "id": 1, "name": "My Timeline", "start_year": 2020, "end_year": 2070 },
  "milestones": [...],
  "notes":      [...]
}
```

---

# Node2 — <NODE2_IP>

**Hardware:** MacBook Pro 2019, Intel i9-9980HK, 64GB RAM, AMD Radeon Pro 5500M 4GB
**OS:** macOS 26.3.1
**Agents:** <AGENT3> (data tracking), <AGENT4> (data processing), <AGENT5> (document extraction service)

---

## agent-central — Server Status API (Node2)
**Port:** `6000`  **Base:** `http://<NODE2_IP>:6000`

| Method | Endpoint  | Description                              |
|--------|-----------|------------------------------------------|
| GET    | `/status` | Server state, agent threads, queue depth |
| GET    | `/queue`  | Current dispatcher queue size            |

`/status` response:
```json
{
  "server": "agent-central",
  "cooldown_sec": 30,
  "queue_pending": 0,
  "agents": {"<AGENT3>": {"alive": true}, "<AGENT4>": {"alive": true}, "<AGENT5>": {"alive": true}},
  "stopping": false
}
```

---

## <AGENT3> — Data Tracking Agent API
**Port:** `6001`  **Base:** `http://<NODE2_IP>:6001`  **Model:** `phi4:10t` (14B, 10 threads)  **Priority:** 1

| Method | Endpoint   | Description                              |
|--------|------------|------------------------------------------|
| GET    | `/status`  | Agent state, model, history length, queue |
| POST   | `/message` | Submit text message → queued, non-blocking |
| POST   | `/clear`   | Clear conversation history               |

`/message` body: `{"text": "your message"}`
`/status` response:
```json
{
  "agent": "<AGENT3>",
  "model": "phi4:10t",
  "history": 12,
  "queue_size": 0,
  "running": true
}
```
Replies are sent to the configured Telegram chat (`AGENT3_CHAT_ID`).

---

## <AGENT4> — Data Processing Agent API
**Port:** `6002`  **Base:** `http://<NODE2_IP>:6002`  **Model:** `qwen2.5-coder:1.5b`  **Priority:** 0 (highest)

| Method | Endpoint   | Description                                    |
|--------|------------|------------------------------------------------|
| GET    | `/status`  | Agent state, model, Gmail status, queue        |
| POST   | `/message` | Submit text message → queued, non-blocking     |
| POST   | `/task`    | Submit structured data processing task         |
| POST   | `/clear`   | Clear conversation history                     |

`/message` body: `{"text": "your message"}`
`/task` body:
```json
{
  "brief":    "Process bank statements for Q1 2020",
  "task_type": "bookkeeping",
  "data":     {"account": "checking", "period": "2020-Q1"}
}
```
`/status` response:
```json
{
  "agent": "<AGENT4>",
  "model": "qwen2.5-coder:1.5b",
  "gmail": "not configured",
  "queue_size": 0,
  "running": true
}
```
Replies sent to Telegram chat (`AGENT4_CHAT_ID`). Reports sent for Telegram approval before submission.

---

## <AGENT5> — Document Extraction Service API
**Port:** `6003`  **Base:** `http://<NODE2_IP>:6003`  **Model:** `deepseek-r1:1.5b`  **Priority:** N/A (bypasses dispatcher)

| Method | Endpoint   | Description                              |
|--------|------------|------------------------------------------|
| GET    | `/status`  | Service state, model, stats              |
| POST   | `/extract` | Extract text from document → structured JSON |

`/extract` body:
```json
{
  "file_id": "telegram_file_id",
  "file_name": "document.pdf",
  "source": "telegram",
  "structure": true,
  "doc_type": "type_a",
  "agent": "<AGENT3>"
}
```

Parameters:
- `file_id` (required): Telegram file ID or local file path
- `file_name` (required): Original filename
- `source` (required): "telegram" or "local"
- `structure` (optional): Enable AI-based structuring/chunking (default: false)
- `doc_type` (optional): Document type for context-aware extraction (e.g., "type_a", "type_b")
- `agent` (optional): Requesting agent name for token routing ("<AGENT3>", "<AGENT4>")

`/extract` response (structure=false):
```json
{
  "status": "success",
  "text": "extracted content...",
  "pages": 5,
  "method": "pymupdf",
  "char_count": 12450,
  "processing_time_ms": 450
}
```

`/extract` response (structure=true):
```json
{
  "status": "success",
  "mode": "structured",
  "char_count": 12450,
  "structured": {
    "chunk_count": 14,
    "processing_method": "900-char chunking + deepseek-r1:1.5b",
    "chunks": [...]
  },
  "processing_time_ms": 240000
}
```

`/status` response:
```json
{
  "agent": "<AGENT5>",
  "model": "deepseek-r1:1.5b",
  "extractions": 42,
  "uptime_hours": 24.5,
  "running": true
}
```

**Architecture (Production Ready):**
- Stateless service (no memory/history, bypasses dispatcher)
- Multi-agent support: Uses requesting agent's bot token for Telegram downloads
- Extraction: PyMuPDF → PyPDF2 → OCR fallback
- Chunking: 900-char optimal chunks for 1.5B model processing
- Returns structured JSON to requesting agent for domain-specific formatting
- Used by <AGENT3> and <AGENT4> to offload document processing

**Performance:**
- Typical PDF extraction: ~4 minutes (with structure=true)
- Time savings: 30+ minutes vs 14B models doing extraction + formatting
- Quality tradeoff: 1.5B extraction is faster but lower quality than 14B
- Architecture: <AGENT5> extracts → agent formats with specialized model

---

## Dispatcher Notes

**Node1:**
- One job runs at a time across all agents (shared queue)
- <AGENT2> priority = 0 (processed first), <AGENT1> priority = 1
- Default cooldown between tasks: 30s (env: `AGENT_COOLDOWN`)
- <AGENT1> → <AGENT2> handoff: POST to `http://<NODE1_IP>:5002/task`

**Node2:**
- One job runs at a time across <AGENT3>/<AGENT4> (shared queue)
- <AGENT4> priority = 0 (processed first), <AGENT3> priority = 1
- <AGENT5> bypasses dispatcher entirely (stateless API service, processes requests immediately)
- Default cooldown between tasks: 10s (env: `AGENT_COOLDOWN`)
- <AGENT3> → <AGENT4> handoff: POST to `http://<NODE2_IP>:6002/task`
- <AGENT3>/<AGENT4> → <AGENT5> extraction: POST to `http://<NODE2_IP>:6003/extract`
  - <AGENT5> uses requesting agent's bot token (pass `"agent": "<AGENT3>"` or `"agent": "<AGENT4>"`)
  - Returns structured JSON with 900-char chunks for agent-specific formatting

---

## Quick Status Check

**Node1:**
```bash
curl http://<NODE1_IP>:5000/status
curl http://<NODE1_IP>:5001/status
curl http://<NODE1_IP>:5002/status
curl -H "X-API-Key: <key>" http://<NODE1_IP>:8000/api/timeline/current
```

**Node2:**
```bash
curl http://<NODE2_IP>:6000/status
curl http://<NODE2_IP>:6001/status
curl http://<NODE2_IP>:6002/status
curl http://<NODE2_IP>:6003/status
```
