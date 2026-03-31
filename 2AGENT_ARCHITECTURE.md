# Agent Architecture - What Works & What Doesn't

Living document tracking lessons learned while building agent-central.

---

## 🤖 Agent Team Overview

### Node1 (192.168.1.10)

**<AGENT1>** - Telegram Chat Agent
- **Port:** 5001
- **Model:** qwen2.5:1.5b
- **Priority:** 1 (lower)
- **Purpose:** General chat and personal assistant tasks via Telegram
- **Capabilities:** Conversation, task management, quick queries
- **Interface:** HTTP API + Telegram bot

**<AGENT2>** - Email Campaign Agent
- **Port:** 5002
- **Model:** qwen2.5:3b
- **Priority:** 0 (highest on Node1)
- **Purpose:** Email outreach campaigns and Gmail automation
- **Capabilities:** Draft emails, send campaigns, manage leads, Gmail integration
- **Approval Flow:** Sends drafts to Telegram for approval before sending
- **Interface:** HTTP API (`/task` endpoint for structured campaigns) + Telegram bot

### Node2 (192.168.1.20)

**<AGENT5>** - Document Extraction Service 🚀 NEW (March 2026)
- **Port:** 6003
- **Model:** deepseek-r1:1.5b
- **Priority:** N/A (bypasses dispatcher - stateless service)
- **Purpose:** Fast document extraction and chunking service for other agents
- **Capabilities:**
  - Extract text from PDFs and documents
  - Chunk text at 900 characters (optimal for 1.5B models)
  - Return structured JSON to requesting agents
  - Multi-agent support (uses requesting agent's bot token)
- **Interface:** HTTP API only (no Telegram)
- **Performance:** 4-minute extraction for typical PDFs
- **Used by:** <AGENT3> and <AGENT4> for PDF processing

**<AGENT3>** - Health Tracking Agent
- **Port:** 6001
- **Model:** phi4:latest (14B) ⚠️ Recently upgraded from qwen2.5-coder:7b
- **Priority:** 1 (lower)
- **Purpose:** Personal health data tracking and analysis
- **Capabilities:**
  - Process medical PDFs via <AGENT5> (bloodwork, lab results)
  - Track weight, vitals, medications, symptoms
  - Analyze health trends over time
  - Maintain structured health records (BLOODWORK.md, WEIGHT.md, GOALS.md)
- **Interface:** HTTP API + Telegram bot
- **Data Location:** `core/agents/<AGENT3>/workspace/metrics/`
- **PDF Flow:** Receives PDF → Routes to <AGENT5> → Formats <AGENT5>'s JSON output

**<AGENT4>** - Financial Operations Agent
- **Port:** 6002
- **Model:** qwen2.5-coder:1.5b ⚠️ (APIHUB shows qwen3:14b - outdated)
- **Priority:** 0 (highest on Node2)
- **Purpose:** Financial bookkeeping and transaction management
- **Capabilities:** Process bank statements via <AGENT5>, track expenses, financial reporting, Gmail integration
- **Approval Flow:** Sends reports to Telegram for approval before submission
- **Interface:** HTTP API (`/task` endpoint for structured financial tasks) + Telegram bot
- **PDF Flow:** Receives PDF → Routes to <AGENT5> → Formats <AGENT5>'s JSON output

### Infrastructure

**Dispatcher Pattern (Both Machines):**
- Single shared queue per machine (one job at a time)
- Priority 0 agents processed first (<AGENT2>, <AGENT4>)
- 10s cooldown between tasks (configurable via `AGENT_COOLDOWN`)
- Non-blocking API submission (jobs queued, return immediately)

**Cross-Agent Handoffs:**
- Node1: <AGENT1> → <AGENT2> via POST to `http://192.168.1.10:5002/task`
- Node2: <AGENT3> → <AGENT4> via POST to `http://192.168.1.20:6002/task`

**Communication:**
- All agents → Telegram for user interaction
- Agents → HTTP APIs for programmatic access
- Server status: `/status` endpoints on ports 5000 (Node1), 6000 (Node2)

### Agent Specialization & Use Cases

**When to use <AGENT1>:**
- Quick questions and general chat
- Task reminders and note-taking
- Lightweight queries that need fast responses
- Non-critical conversational tasks

**When to use <AGENT2>:**
- Email campaigns and outreach
- Lead management and CRM tasks
- Gmail automation
- Marketing and sales operations
- **Requires approval** before sending emails

**When to use <AGENT3>:**
- Health data tracking and analysis
- Processing medical PDFs (bloodwork, prescriptions)
- Logging weight, vitals, symptoms, medications
- Health trend analysis over time
- Medical record organization

**When to use <AGENT4>:**
- Financial bookkeeping and accounting
- Bank statement processing
- Expense tracking and reporting
- Financial analysis and budgeting
- **Requires approval** before submitting reports

**Priority System Rationale:**
- **Node1:** <AGENT2> (email/business) > <AGENT1> (personal chat)
- **Node2:** <AGENT4> (financial/critical) > <AGENT3> (health tracking)
- Higher priority agents process time-sensitive business operations first
- Lower priority agents handle less urgent personal tasks

---

## ✅ What Works

### Models

**phi4:latest (14B)**
- Tool calling: Excellent - follows parameter names correctly
- Quality: High-quality responses, good understanding
- PDF processing: Handles complex medical documents well
- Drawback: Slow (2-15 min for complex tasks)
- Best for: Quality-critical tasks, complex reasoning

**qwen2.5-coder:1.5b**
- Speed: Very fast responses
- Best for: Simple tasks, low-priority agents
- Context window: Limited, keep prompts tight

### PDF Processing with <AGENT5> Routing (PRODUCTION READY - March 2026)

**Architecture Overview:**
- **<AGENT5> Service:** Centralized extraction service on port 6003
- **Model:** deepseek-r1:1.5b (fast extraction, 4-minute processing)
- **Flow:** Agent receives PDF → Routes to <AGENT5> → Gets structured JSON → Formats with agent's model

**PyMuPDF (fitz) - PRIMARY EXTRACTION METHOD**
- Most robust extraction method
- Handles medical documents, lab results reliably
- Fast and local (no API)
- Extracts ~5500 chars cleanly from bloodwork PDFs

**Multi-layer fallback works:**
```
PyMuPDF → PyPDF2 → OCR (if image-based)
```

**Chunking Strategy with <AGENT5>:**
- **<AGENT5> (1.5b):** 900 char chunks (proven optimal from VAIO testing)
  - Processes entire document in ~4 minutes
  - Returns structured JSON to requesting agent
- **Agent formatting:** Uses agent's model (1.5b-14B) for domain-specific formatting
  - <AGENT4> (1.5b): 2-minute formatting
  - <AGENT3> (14B): 21-minute high-quality formatting
- **Total time savings:** 30+ minutes vs 14B models doing everything

**Critical Quality vs Speed Tradeoff:**
- **KNOWN LIMITATION:** <AGENT5>'s 1.5B extraction is lower quality than 14B
- **Accepted tradeoff:** Fast extraction + quality formatting
- **Why it works:** Agents still use their specialized models for final output
- **Future optimization:** Upgrade <AGENT5> model when better small models available

**Implementation Details:**
- <AGENT5> uses requesting agent's bot token for Telegram downloads
- Agents pass `"agent": AGENT_NAME` parameter to <AGENT5>
- 30-minute timeouts (1800s) for large documents
- Both <AGENT3> and <AGENT4> successfully routing through <AGENT5>

### Context Management

**Smart history truncation (CRITICAL)**
- Truncate assistant responses to 800 chars in history
- Keep user messages intact
- Prevents context explosion from long responses
- Without this: 14B model times out (15+ min)

**Context-aware file injection:**
- Skip injection for simple write requests ("write to", "create the")
- Skip injection when history >5K chars
- Prevents overwhelming model with redundant data

**Telegram message chunking:**
- Auto-split messages >4096 chars
- Add "..." markers between chunks
- Prevents silent failures

### Tool Calling

**Tool-first system prompt structure:**
```
1. Tool usage instructions (PRIORITY #1, top of prompt)
2. Agent identity (SOUL.md)
3. Behavior rules
4. File reading instructions
```

**Critical instructions that work:**
- "Call tools FIRST, then add brief confirmation"
- "Do NOT generate long explanations before tool calls"
- Explicit parameter documentation: `write_file uses 'path' parameter`

**write_file() design:**
- Accept both `path` and `filename` parameters
- Backward compatibility prevents breakage
- Clear error messages on failure

**JSON parsing improvements:**
- Strip control characters (\n, \r, \t) before parsing
- Collapse multiple spaces
- Debug logging for failed parses
- Handles malformed JSON from model responses

### Timeouts

**Current working values:**
- Main chat: 900s (15 min)
- PDF chunk summarization: 300s (5 min)
- Warmup: 900s

### Agent Configuration

**Timezone awareness:**
- pytz with America/New_York
- Shows "Friday, March 28, 2026 at 8:15 PM EST"
- Critical for health tracking with timestamps

**Character limits:**
- File injection: 6000 chars
- PDF extraction: 6000 chars
- Works well for medical documents

---

## ❌ What Doesn't Work (FIXED)

### Models

**qwen2.5-coder:7b**
- Tool calling: UNRELIABLE - uses wrong parameter names
- Example failure: Used `file_name=` instead of `path=`
- Hallucinates success when tools fail silently
- Not following system prompt instructions
- **Verdict:** Don't use for production tool-heavy workflows

### File Prefetch (FIXED 2026-03-29)

**Old Problem:** Messages about bloodwork/metrics classified as "tasks" topic
- Would load TASKS.md instead of BLOODWORK.md, WEIGHT.md, GOALS.md
- Health keywords missing: blood, bloodwork, lab, metrics, test results
- Ignored workspace subdirectories (metrics/)

**Fixed:**
- Added 12 health-related keywords to topic detection
- Health topic now auto-loads ALL files from metrics/ directory
- Enhanced path mining to check workspace/metrics/ for <AGENT3>
- Reduced task keyword false positives

### Context Issues

**Long responses in history without truncation:**
- Caused 15 min timeouts with 14B model
- 10K+ char context → model chokes
- PDF summaries (2K+ chars) stay in full

**Always injecting 6000 chars:**
- Ignores current context size
- Adds unnecessary data for simple write requests
- User already has data from previous messages

**Tool instructions buried in system prompt:**
- Model skips over them
- Generates long explanations before tool calls
- Takes forever to get to actual work

### Tool Calling Failures

**Silent failures:**
- Model calls tool with wrong parameters
- Tool fails with error
- Model doesn't see the error
- Model tells user "success" (hallucination)

**Parameter name mismatches:**
- 7B invented `file_name=` (with underscore)
- Function only accepts `path` or `filename` (no underscore)
- System prompt clearly documented `path` parameter
- Model ignored documentation

### PDF Processing Issues

**Timeout with chunk combination:**
- 14B model + large context → slow CPU inference
- Full PDF (5545 chars + context = 8632 chars) times out at 900s (15 min)
- Fixed by: Adaptive chunking (2500 chars for 14B vs 900 for small models)
- Result: 3 chunks instead of 7, balances speed and quality

**PyPDF2 alone:**
- Less robust than PyMuPDF
- Okay as fallback, not as primary

### Core Identity Loading

**⚠️ KNOWN ISSUE (not yet fixed):**
- `_load_core_identity()` hard-coded to 400 chars
- SOUL.md is 3174 chars but only 400 loaded
- Agent gets "lobotomized" version of personality
- Causes short, unhelpful responses
- Need to increase to 3000+ chars

---

## 🔧 Architecture Patterns

### Dispatcher Pattern
- Central queue with priority handling
- <AGENT4>: priority 0 (higher)
- <AGENT3>: priority 1
- 10s cooldown between tasks

### Agent Structure
```
agent-central/
├── bot.py (<AGENT3> agent)
├── core/
│   ├── agents/
│   │   ├── <AGENT3>/
│   │   │   ├── workspace/
│   │   │   │   ├── SOUL.md (identity)
│   │   │   │   ├── MEMORY.md (operator context)
│   │   │   │   ├── TOOLS.md (tool docs)
│   │   │   │   └── skills/ (specialized capabilities)
│   │   └── <AGENT4>/
│   ├── tools/ (write_file, run_command)
│   ├── dispatcher.py
│   └── tools_manager.py (JSON tool parser)
└── server.py (multi-agent orchestrator)
```

### Tool Execution Flow
1. User sends message
2. Agent calls ask_ollama()
3. Model generates response with ```json tool blocks
4. tools_manager.py extracts JSON
5. Dispatches to tool function
6. Tool returns result dict
7. Agent sends confirmation to user

---

## ⚡ Performance Tuning

### Ollama CPU Thread Count

Control inference speed vs system resource usage via `.env`:

```bash
# Default: 8 threads (50% CPU on 16-core system)
# Configured in .env as OLLAMA_NUM_THREAD
```

**Thread Count Guidelines (16-core Intel i9):**

| Threads | CPU Usage | Performance | System Wear | Use Case |
|---------|-----------|-------------|-------------|----------|
| 8 | 50% | Baseline | Low | Background processing, multitasking |
| 10 | 62% | +25% faster | Moderate | Balanced performance (recommended for testing) |
| 12 | 75% | +50% faster | Moderate-High | Active development, frequent inference |
| 14 | 87% | +75% faster | High | Demanding workloads, batch processing |
| 16 | 100% | Maximum | Very High | Maximum speed (may make system sluggish) |

**How to Apply:**

**Method 1: Custom Modelfile (RECOMMENDED - works reliably):**
1. Create Modelfile with thread parameter:
   ```bash
   cat > /tmp/phi4-10t.Modelfile <<EOF
   FROM phi4:latest
   PARAMETER num_thread 10
   EOF
   ```
2. Create custom model:
   ```bash
   ollama create phi4:10t -f /tmp/phi4-10t.Modelfile
   ```
3. Update `.env`:
   ```bash
   AGENT3_OLLAMA_MODEL=phi4:10t
   ```
4. Restart agents: autoreload picks it up automatically

**Method 2: API options (may not work on all Ollama versions):**
- Add `"options": {"num_thread": 10}` to API calls
- Not reliable across all Ollama versions
- Use Method 1 (Modelfile) for guaranteed results

**Verification:**
Watch LLM monitor during inference - should show ~1000% CPU (10 cores) instead of 665% (6-7 cores).

**Performance Impact Example (5545 char PDF):**
- 8 threads (default): ~12-15 min (estimated)
- 10 threads (phi4:10t): ~81 min (actual, 2026-03-29 workflow)
  - Note: Longer than expected - needs investigation
  - Expected was ~10-12 min with 3x2500 char chunks
- 14 threads: ~8-10 min (estimated)

**Trade-offs:**
- Higher threads = faster inference BUT more heat, fan noise, power consumption
- Lower threads = slower BUT cooler system, lower wear, better for long-running ops
- Adjust based on workload: increase for batch jobs, decrease for 24/7 operation

---

## 📝 Best Practices

1. **Use 14B for tool-heavy workflows** - 7B is too unreliable
2. **Always truncate assistant responses in history** - Prevents context explosion
3. **Tool instructions go first in system prompt** - Model pays more attention
4. **Skip file injection for write requests** - User already has the data
5. **Log context size** - Debug timeouts by checking char counts
6. **Multi-layer PDF fallback** - PyMuPDF → PyPDF2 → OCR
7. **Adaptive PDF chunking** - 14B: 2500 char chunks (CPU inference time), small models: 900 char chunks
8. **Preprocess heavy I/O** - Machine extracts PDFs before agent reasoning (parallel processing)
9. **CPU inference awareness** - Longer context = exponentially slower on CPU, chunk accordingly
10. **Use pytz for timezone** - Agent needs accurate time awareness
11. **Message chunking for Telegram** - Split at 4096 chars automatically
12. **Accept multiple parameter names** - `path` and `filename` for backward compat
13. **Use autoreload during development** - `./start --watch` for instant code changes
14. **Strip control chars in JSON parsing** - Models add newlines/tabs that break JSON.loads()

---

## 🐛 Known Issues

1. **Core identity truncated to 400 chars** - Should be 3000+
2. **7B model tool calling unreliable** - Invents parameter names
3. **Silent tool failures** - Model doesn't see errors, hallucinates success
4. **10-thread performance slower than expected** - 2026-03-29 actual workflow:
   - Expected: ~11-13 min (3 chunks @ 2500 chars, 10 threads)
   - Actual: 81 min (6x slower)
   - Possible causes: timeout/retries, queue delays, thread config not applied correctly
   - Need to investigate: check logs for timeouts, verify CPU usage during full workflow

---

## 📊 Performance Metrics

| Agent | Task | Model | Time | Success |
|-------|------|-------|------|---------|
| <AGENT3> | Simple text response | phi4:14B | ~20s | ✅ |
| <AGENT3> | Simple text response | qwen2.5-coder:7b | ~3s | ✅ |
| <AGENT3> | PDF extraction (5500 chars) | phi4:14B | ~10s | ✅ |
| <AGENT3> | PDF processing (5500 chars, 7 x 900 char chunks) | phi4:14B | ~20 min | ✅ |
| <AGENT3> | PDF processing (5500 chars, full context) | phi4:14B | TIMEOUT | ❌ (15 min CPU limit) |
| <AGENT3> | Full workflow: PDF → extract → analyze → write (3 x 2500 char chunks, 10 threads) | phi4:10t | 81 min | ✅ (needs optimization) |
| <AGENT3> | Write to file (simple) | phi4:14B | ~30s | ✅ (with context optimization) |
| <AGENT3> | Write to file (simple) | qwen2.5-coder:7B | TIMEOUT | ❌ (wrong params) |
| <AGENT3> | Write to file (after PDF) | phi4:14B | 15 min → TIMEOUT | ❌ (fixed with truncation) |

---

**Last Updated:** 2026-03-29
**Agent Version:** agent-central master branch
**Active Machines:**
- Node1 (192.168.1.10): <AGENT1> (qwen2.5:1.5b), <AGENT2> (qwen2.5:3b)
- Node2 (192.168.1.20): <AGENT3> (phi4:14B), <AGENT4> (qwen2.5-coder:1.5b)

**Recent Changes:**
- 2026-03-29: Upgraded <AGENT3> from qwen2.5-coder:7b → phi4:14B for reliable tool calling
- 2026-03-29: Added context management optimizations (history truncation, smart injection)
- 2026-03-29: Increased timeouts to 900s for 14B model processing
- 2026-03-29: Adaptive PDF chunking (2500 chars for 14B, balances CPU inference time vs quality)
