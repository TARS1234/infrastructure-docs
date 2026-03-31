# MONITORING.md — Infrastructure Monitoring Strategy

**Purpose:** Complete monitoring strategy for multi-agent AI infrastructure. Essential operational knowledge for maintaining fleet health, detecting bottlenecks, and preventing service degradation.

**Last Updated:** 2026-03-31

---

## Why Monitoring is Essential

### The Problem: Invisible Performance Degradation

Running local LLMs and multi-agent systems creates unique operational challenges:

1. **Silent resource exhaustion**
   - Models consume 10-30GB+ RAM per instance
   - Without monitoring: first sign of trouble is OOM killer terminating agents
   - By then: conversation history lost, tasks interrupted, user experience degraded

2. **CPU inference bottlenecks**
   - 14B models on CPU take 2-20+ minutes per task
   - Context size directly impacts inference time (exponential on CPU)
   - Without monitoring: can't distinguish "agent is working" from "agent is stuck"

3. **Thermal throttling**
   - Heavy inference → sustained 80-100% CPU → 90°C+ temps
   - CPU throttles → inference slows down → tasks timeout
   - Without monitoring: mysterious timeouts with no obvious cause

4. **Swap thrashing**
   - Multiple models + limited RAM → system swaps to disk
   - Model inference becomes 10-100x slower (disk I/O vs RAM)
   - Without monitoring: agents appear "frozen" but are actually disk-bound

5. **Multi-agent queue congestion**
   - Dispatcher queues tasks, one runs at a time
   - Priority 0 agents (<AGENT4>, <AGENT2>) block priority 1 agents (<AGENT3>, <AGENT1>)
   - Without monitoring: can't see queue depth or which agent is blocking

### The Solution: Real-Time Visibility

**llm-monitor** provides:
- **Proactive detection**: See resource pressure before failure
- **Performance tuning**: Identify bottlenecks (CPU, RAM, swap, thermal)
- **Agent accountability**: Which agent is using which model and how much CPU/RAM
- **Capacity planning**: Know when to upgrade hardware or distribute workload
- **Debugging context**: Correlate agent behavior with system metrics

---

## What to Monitor

### Critical Metrics

#### 1. CPU Utilization & Frequency
**Why it matters:**
- Ollama uses 8-16 threads per model (configurable)
- High CPU = active inference, low CPU = idle/loaded
- Frequency drops = thermal throttling

**What to watch:**
- Overall CPU: sustained >85% = bottleneck risk
- Per-core: unbalanced load = inefficient threading
- Frequency: drops below base clock = throttling
- Load average: >core count = queue buildup

**Fleet baselines:**
| Machine | Cores | Base Freq | Normal Load | Heavy Load | Throttle Temp |
|---------|-------|-----------|-------------|------------|---------------|
| Node2   | 16    | 2.4 GHz   | 0.5-2.0     | 8.0-16.0   | 100°C         |
| Node6   | 10    | 3.2 GHz   | 0.3-1.5     | 5.0-10.0   | 100°C         |
| Node3   | 8     | 2.8 GHz   | 0.2-1.0     | 4.0-8.0    | 95°C          |
| Node4   | 4     | 2.6 GHz   | 0.1-0.5     | 2.0-4.0    | 90°C          |
| Node5   | 2     | 2.0 GHz   | 0.1-0.3     | 1.0-2.0    | 85°C          |

#### 2. Memory (RAM) Pressure
**Why it matters:**
- Models load into RAM (qwen2.5:1.5b = 1GB, phi4:14B = 9GB, qwen3:14B = 9GB)
- Multiple agents + models = 20-40GB+ consumed
- When RAM fills → swap usage → massive performance hit

**What to watch:**
- Used/Available: <2GB available = danger zone
- Swap usage: >10% = warning, >20% = critical (disk thrashing)
- Wired (macOS): memory locked by kernel, can't be paged
- Compressed (macOS): efficient RAM usage, normal to see 2-5GB

**Fleet baselines:**
| Machine | RAM   | Safe Usage | Warning | Critical | Swap Limit |
|---------|-------|------------|---------|----------|------------|
| Node2   | 64GB  | <55GB      | 58GB    | 62GB     | <2GB       |
| Node6   | 32GB  | <28GB      | 30GB    | 31GB     | <1GB       |
| Node3   | 16GB  | <14GB      | 15GB    | 15.5GB   | <500MB     |
| Node4   | 16GB  | <14GB      | 15GB    | 15.5GB   | <500MB     |
| Node5   | 12GB  | <10GB      | 11GB    | 11.5GB   | <500MB     |

#### 3. GPU Utilization (where applicable)
**Why it matters:**
- Node2 AMD Radeon Pro 5500M (4GB GDDR6) - available but not currently used
- Node6 M1 Max 24-core GPU - best GPU in fleet for ML inference
- Node3 AMD R9 M370X (2GB) - light GPU workloads only

**What to watch:**
- GPU active %: shows if GPU acceleration is working
- GPU power: high power = active compute
- VRAM usage: filling VRAM = model too large for GPU

**Note:** Currently all agents use CPU inference via Ollama. GPU acceleration planned for future.

#### 4. Temperatures
**Why it matters:**
- Sustained inference = sustained heat
- High temps → thermal throttling → slower inference → timeouts
- Fan noise = indicator of thermal stress

**What to watch:**
- CPU die temp: >90°C = throttling likely
- GPU die temp: >85°C = throttling likely
- Sustained high temps: consider better cooling or lower thread count

#### 5. Ollama Process Metrics
**Why it matters:**
- Each runner process = one loaded model
- CPU % shows active inference vs idle
- RSS shows memory footprint per model

**What to watch:**
- **Status:**
  - `active` (CPU >5%, green) = currently inferring
  - `loaded` (CPU <5%, cyan) = warm in RAM, ready
  - `sleeping` (dim) = idle/background
- **RSS Memory:**
  - qwen2.5:1.5b ≈ 1.0-1.5 GB
  - qwen2.5-coder:7b ≈ 4.5-5.0 GB
  - phi4:14B ≈ 9.0-10.0 GB
  - qwen3:14B ≈ 9.0-10.0 GB
  - deepseek-r1:1.5b ≈ 1.0-1.5 GB
- **Model → Agent mapping:**
  - Shows which agent is using which model
  - Example: `qwen:1.5b (<AGENT4>)` = <AGENT4> agent using qwen2.5-coder:1.5b

#### 6. Agent Process Metrics
**Why it matters:**
- Agent CPU shows Python overhead (tool execution, API calls, parsing)
- Agent RSS shows workspace memory (history, files, state)
- Working directory confirms which project agent is operating on

**What to watch:**
- **Coding agents** (Claude Code, Aider):
  - Normal CPU: 2-10% (mostly idle, waiting for LLM)
  - High CPU: 20-50% (file operations, git, npm, docker)
  - RSS: 0.5-2.0 GB typical
- **Custom agents** (multi-agent system):
  - server.py process = orchestrator running multiple agents
  - CPU spike = agent dispatching task or processing LLM response
  - RSS growth = conversation history accumulation

---

## Monitoring Tools

### Primary Tool: llm-monitor

**Location:** `~/llm-monitor/`

**Purpose:** Real-time terminal dashboard showing system + Ollama + agent metrics in one view.

**Installation:**
```bash
cd ~/llm-monitor
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
chmod +x monitor
```

**Usage:**
```bash
./monitor                   # auto-launches with sudo on macOS for full metrics
./monitor --interval 2      # refresh every 2 seconds (faster updates)
./monitor --no-ollama       # skip Ollama process scanning
./monitor --no-claude       # skip Claude Code/Aider scanning
```

**macOS sudo requirement:**
- Without sudo: CPU, RAM, swap, disk I/O only
- With sudo: + GPU utilization, ANE power, thermals, per-component power draw
- **Always run with sudo for full visibility**

**Key Panels:**
1. **CPU** - Overall %, per-core %, frequency, load average, P/E core split, temp, power
2. **GPU / ANE / Power** - GPU util/freq/temp/power, ANE power, package total power
3. **Memory** - Used/available/wired/compressed, swap usage with high-swap warning
4. **Ollama Processes** - PID, CPU%, RSS, Status, Model (with agent name)
5. **Coding Agents** - Tool, CPU%, RSS, Status, Model, Working Dir
6. **Other AI Agents** - Tool, PID, CPU%, RSS, Model, Status
7. **System** - Disk I/O, process count, uptime, platform

**Agent-Central Integration:**
- Reads `~/agent-central/state/ollama/agents.json`
- Maps Ollama models → agent names
- Example display: `qwen:1.5b (<AGENT4>)` = <AGENT4> using qwen2.5-coder:1.5b
- Shows which custom agent is active

### Secondary Tools

**1. Activity Monitor (macOS)**
- Use for: Quick spot checks, memory pressure graph, energy impact
- Location: `/Applications/Utilities/Activity Monitor.app`
- View → All Processes, sort by CPU or Memory

**2. htop (Linux)**
```bash
# Node4 (Arch), Node5 (Xubuntu)
htop
# Press F5 for tree view, F6 to sort
```

**3. tmux sessions (Node2 primary)**
```bash
# Check running services
tmux ls

# Attach to Ollama
tmux attach -t ollama

# Attach to agent runtime
tmux attach -t agent

# Detach: Ctrl+B, then D
```

**4. Agent API status endpoints**
```bash
# Node1 status check
curl http://<NODE1_IP>:5000/status  # agent-central dispatcher
curl http://<NODE1_IP>:5001/status  # <AGENT1>
curl http://<NODE1_IP>:5002/status  # <AGENT2>

# Node2 status check
curl http://<NODE2_IP>:6000/status  # agent-central dispatcher
curl http://<NODE2_IP>:6001/status  # <AGENT3>
curl http://<NODE2_IP>:6002/status  # <AGENT4>
curl http://<NODE2_IP>:6003/status  # <AGENT5>
```

Response shows: agent name, model, history length, queue size, running state.

**5. Ollama API**
```bash
# Check loaded models
curl http://localhost:11434/api/ps

# Response shows:
# - model name (e.g., "phi4:10t")
# - size in bytes
# - loaded timestamp
```

---

## Fleet-Specific Monitoring Strategy

### Node2 (Primary Heavy Node) — 64GB RAM, i9, AMD GPU

**Role:** Main AI brain, orchestrator, multi-agent runtime

**Critical services:**
- Ollama (tmux: ollama) - model serving
- API server (port 5001) - main API gateway
- Telegram bot (tmux: agent) - user interface
- Agent workspace - <AGENT3>, <AGENT4>, <AGENT5>

**Monitoring priorities:**
1. **RAM usage** - 64GB capacity, typically running 2-4 models simultaneously
   - Normal: 30-45GB used (OS + models + agents)
   - Warning: >55GB used (limited headroom)
   - Critical: >60GB used or swap >2GB (performance degradation imminent)

2. **CPU utilization** - 16 threads, expect 50-100% during inference
   - Normal: 20-60% average (idle between tasks)
   - Heavy: 80-100% sustained (active 14B model inference)
   - Check: Per-core distribution (should use 8-14 cores during inference)

3. **Thermal throttling** - Intel i9 runs hot under sustained load
   - Normal: 60-80°C during inference
   - Warning: 85-95°C (fans at max)
   - Critical: >95°C (throttling active, expect 20-30% slower inference)

4. **Ollama process status:**
   - phi4:10t (14B, 10 threads) → <AGENT3>
   - qwen2.5-coder:1.5b → <AGENT4>
   - deepseek-r1:1.5b → <AGENT5>
   - Watch for: Multiple runners loaded = high RAM usage

5. **Agent queue depth:**
   - Check: `curl http://<NODE2_IP>:6000/status | jq .queue_pending`
   - Normal: 0-1 pending
   - Warning: 2-5 pending (agents waiting for dispatcher)
   - Action: If stuck >5 min, check which agent is blocking (priority 0 agents process first)

**Baseline metrics (Node2):**
```
CPU: 20-40% idle, 80-100% active inference
RAM: 30-45GB normal, <55GB safe, >60GB danger
Swap: 0-500MB normal, >2GB = thrashing
Temp: 60-80°C normal, >90°C = throttling
Load Avg: 2-8 normal, >12 = queue buildup
```

### Node6 (Mobile Heavy Node) — 32GB RAM, M1 Max, 24-core GPU

**Role:** Mobile dev machine, high-end execution layer, backup orchestrator

**Monitoring priorities:**
1. **Unified memory** - M1 Max shares 32GB between CPU and GPU
   - Normal: 18-25GB used (OS + dev tools + models)
   - Warning: >28GB used
   - Critical: >30GB used or swap >1GB

2. **GPU utilization** - 24-core GPU with Metal 4, best GPU in fleet
   - Currently unused (CPU inference only)
   - Future: Monitor GPU active % when GPU acceleration enabled

3. **ANE (Apple Neural Engine)** - On-chip ML acceleration
   - Monitor ANE power draw (requires sudo)
   - Normal: 0-2W idle, 5-10W active
   - Indicates on-device ML workloads (future optimization target)

4. **Thermal efficiency** - Apple Silicon thermal management
   - Normal: 40-60°C (M1 runs cooler than Intel)
   - Warning: >70°C
   - M1 Max throttles less aggressively than Intel

**Baseline metrics (Node6):**
```
CPU: 15-30% idle, 60-90% active inference
RAM: 18-25GB normal, <28GB safe, >30GB danger
Swap: 0-200MB normal, >1GB = thrashing
Temp: 40-60°C normal, >70°C = warm
GPU: 0-5% idle (future: 40-80% during GPU inference)
ANE: 0-2W idle, 5-10W active
```

### Node3 (Security/Utility Node) — 16GB RAM, i7, AMD R9 M370X

**Role:** Network security, log aggregation, backup coordination

**Monitoring priorities:**
1. **RAM constraint** - Only 16GB, can run 1-2 small models max
   - Normal: 8-12GB used
   - Warning: >14GB used
   - Critical: >15GB used (no headroom for model loading)

2. **Storage space** - 500GB SSD, 358GB free
   - Monitor: `/` usage for logs and backups
   - Warning: <50GB free

3. **macOS Monterey compatibility** - Older OS version
   - Check: Software compatibility when deploying new agents
   - Note: Max supported macOS for this hardware

**Baseline metrics (Node3):**
```
CPU: 10-20% idle, 40-70% active
RAM: 8-12GB normal, <14GB safe, >15GB danger
Disk: 358GB free, >50GB minimum
Temp: 50-70°C normal
```

### Node4 (Lightweight Worker) — 16GB RAM, i7, Arch Linux

**Role:** Edge compute, remote deployment, Tailscale jump host

**Monitoring priorities:**
1. **Low-resource footprint** - Designed for stateless tasks
   - Normal: 3-6GB RAM used
   - Keep: <12GB used to maintain low footprint

2. **Disk space constraint** - Only 27GB free on root
   - Critical: Don't store large models here
   - Use: Streaming inference or remote model access

3. **Tailscale connectivity** - Always accessible at <TAILSCALE_IP>
   - Monitor: `tailscale status` for mesh connectivity
   - Check: Can reach from other nodes

4. **Bleeding-edge packages** - Arch rolling release
   - Watch: Package updates breaking agent dependencies
   - Test: New Ollama/Python versions here first

**Baseline metrics (Node4):**
```
CPU: 5-15% idle, 30-60% active
RAM: 3-6GB normal, <12GB safe, 15GB max
Disk: 27GB free (limited, don't store models)
Load: 0.2-1.0 normal (low baseline)
```

### Node5 (Storage Node) — 12GB RAM, i7, 1TB HDD

**Role:** Bulk storage, backup host, low-intensity automation

**Monitoring priorities:**
1. **HDD performance** - 1TB HDD (not SSD), slower I/O
   - Monitor: Disk I/O MB/s (HDD = 80-120 MB/s max)
   - Avoid: Heavy random I/O workloads
   - Good for: Sequential writes (backups, logs, archives)

2. **RAM limitation** - Only 12GB, smallest in fleet
   - Normal: 6-9GB used
   - Max: 1 small model only (qwen2.5:1.5b)

3. **Storage capacity** - 859GB free (best for bulk storage)
   - Use: Model archives, backup staging, log storage
   - Monitor: Growth rate of backup directory

**Baseline metrics (Node5):**
```
CPU: 5-10% idle, 20-40% active
RAM: 6-9GB normal, <10GB safe, 11GB max
Disk: 859GB free (HDD, slower I/O)
Disk I/O: 20-40 MB/s normal, 80-120 MB/s max
```

---

## Agent Monitoring Best Practices

### 1. Correlation: System Metrics ↔ Agent Behavior

**Scenario: Agent timeout after 15 minutes**

**Without monitoring:**
- "Agent stuck, need to restart"
- No idea why, just know it timed out

**With monitoring:**
```
Time: T+0      → Task submitted, agent starts inference
Time: T+2 min  → CPU spikes to 100%, RAM at 45GB, normal
Time: T+8 min  → CPU still 100%, temp hits 95°C, fan maxed
Time: T+10 min → CPU freq drops 2.4GHz → 1.8GHz (throttling!)
Time: T+15 min → Timeout (inference 40% slower due to throttling)
```

**Root cause:** Thermal throttling → slower inference → timeout

**Solution:** Reduce Ollama thread count (16 → 10 threads) or improve cooling

### 2. Model Sizing: RAM → Model Selection

**Agent configuration decision:**

```python
# Bad: Always use largest model
AGENT_MODEL = "phi4:latest"  # 14B = 9GB RAM per instance

# Problem with 3 agents on 32GB machine:
# - 3 × 9GB = 27GB for models alone
# - OS + agents + workspace = 8GB
# - Total: 35GB > 32GB available
# - Result: Swap thrashing, 10x slower inference

# Good: Size models to available RAM
AGENT3_MODEL = "phi4:10t"            # 14B, priority tasks, 9GB
AGENT4_MODEL = "qwen2.5-coder:1.5b"  # Fast, priority 0, 1GB
AGENT5_MODEL = "deepseek-r1:1.5b"    # Fast extraction, 1GB

# Total: 9GB + 1GB + 1GB = 11GB models + 8GB system = 19GB < 32GB safe
```

**Monitoring validates:**
- Check RAM usage after loading all 3 models
- Confirm no swap usage during concurrent inference
- Verify expected RSS per Ollama runner process

### 3. Performance Tuning: Thread Count Optimization

**Thread count trade-offs:**

| Threads | CPU % | Inference Speed | Heat | Power | Fan Noise | Wear |
|---------|-------|-----------------|------|-------|-----------|------|
| 8       | 50%   | Baseline        | Low  | Low   | Quiet     | Low  |
| 10      | 62%   | +25% faster     | Med  | Med   | Moderate  | Med  |
| 12      | 75%   | +50% faster     | High | High  | Loud      | High |
| 16      | 100%  | +75% faster     | Max  | Max   | Max       | Max  |

**How to monitor thread count impact:**

1. **Create custom model with thread parameter:**
   ```bash
   cat > /tmp/phi4-10t.Modelfile <<EOF
   FROM phi4:latest
   PARAMETER num_thread 10
   EOF
   ollama create phi4:10t -f /tmp/phi4-10t.Modelfile
   ```

2. **Update agent config:**
   ```bash
   # In agent .env file
   AGENT3_OLLAMA_MODEL=phi4:10t
   ```

3. **Monitor during inference:**
   - Watch: CPU % (should be ~625% = 6.25 cores for 10 threads)
   - Watch: Temp (should stay <90°C)
   - Watch: Inference time (compare to baseline)
   - Watch: Fan noise (subjective but important for 24/7 operation)

4. **Measure results:**
   ```
   8 threads:  12-15 min for 5500 char PDF processing
   10 threads: 10-12 min for same task
   14 threads: 8-10 min for same task
   ```

**Best practice:** Use 10 threads on 16-core systems (62% CPU, good balance)

### 4. Context Management: History Size → Inference Time

**Known issue from agent architecture testing:**

```python
# Bad: Keep full assistant responses in history
history.append({
    "role": "assistant",
    "content": full_response  # 2000+ chars, includes full PDF summaries
})

# Problem:
# - Context size: 12 turns × 2000 chars = 24,000 chars
# - Result: 14B model on CPU = 15+ min inference time or timeout

# Good: Truncate assistant responses in history
history.append({
    "role": "assistant",
    "content": truncate_response(full_response, max_chars=800)
})

# Result:
# - Context size: 12 turns × 800 chars = 9,600 chars
# - Inference time: 2-5 min (acceptable)
```

**Monitoring indicators:**
- Long inference time (>5 min for simple query)
- High CPU sustained for entire duration
- Check agent logs for context size in characters

**Solution validated in 2AGENT_ARCHITECTURE.md:**
- Truncate assistant responses to 800 chars
- Keep user messages intact
- Skip file injection when history >5K chars

### 5. Queue Visibility: Dispatcher Congestion

**Multi-agent dispatcher pattern:**
```
Priority 0 agents (<AGENT4>, <AGENT2>) process first
Priority 1 agents (<AGENT3>, <AGENT1>) wait in queue
One job runs at a time per machine
```

**Without monitoring:**
- User: "<AGENT3> isn't responding"
- Reality: <AGENT4> is processing a 4-minute task, <AGENT3> is queued

**With monitoring:**
```bash
# Check queue status
curl http://<NODE2_IP>:6000/status

{
  "server": "agent-central",
  "cooldown_sec": 10,
  "queue_pending": 2,          # ← 2 tasks waiting
  "agents": {
    "<AGENT3>": {"alive": true},   # ← Currently running
    "<AGENT4>": {"alive": true},
    "<AGENT5>": {"alive": true}
  },
  "stopping": false
}
```

**See in llm-monitor:**
- Ollama panel: Shows which model is `active` (CPU >5%)
- Model name: Maps to agent via agent-central state file
- Example: `phi4:10t (<AGENT3>)` with CPU 85% = <AGENT3> is running, others are queued

**Best practice:**
- Monitor queue depth during heavy usage
- If queue >3, consider priority rebalancing or multi-machine distribution
- Use priority 0 for time-sensitive business operations
- Use priority 1 for background/personal tasks

### 6. Thermal Awareness: Sustained Load Management

**24/7 operation considerations:**

**Scenario: Running agents overnight**
```
Hour 0: Start batch PDF processing (20 documents)
Hour 1: CPU 100%, temp 85°C, fan medium
Hour 2: CPU 100%, temp 95°C, fan max
Hour 3: CPU 75% (throttled!), temp 100°C, fan max
Hour 4: Inference slows 30%, tasks start timing out
Hour 5: Agent crashes (thermal shutdown protection)
```

**Monitoring catches this early:**
- Hour 1: See sustained 100% CPU + rising temp → reduce thread count
- Or: Split workload across Node2 (8 threads) + Node6 (8 threads)
- Or: Add cooldown delays between tasks (30s-60s)

**Best practice for long-running operations:**
1. Monitor temp continuously
2. If temp >85°C sustained, reduce thread count or add delays
3. Consider lower-power models for batch work (1.5B vs 14B)
4. Use cooler machines (M1 Max) for sustained inference
5. Night operations: Use lower thread count (8 threads = cooler, quieter)

---

## Performance Baselines & Expectations

### Model Inference Times (Node2: i9, 64GB, 10 threads)

| Task | Model | Context | Expected Time | Warning Signs |
|------|-------|---------|---------------|---------------|
| Simple chat response | phi4:14B | 2K chars | 20-40s | >60s = check CPU throttling |
| Simple chat response | qwen2.5:1.5b | 2K chars | 3-8s | >15s = check system load |
| PDF extraction | deepseek-r1:1.5b | 5500 chars | 4 min | >8 min = check memory |
| PDF processing (3 chunks) | phi4:10t | 7500 chars | 10-12 min | >20 min = check throttling |
| Full PDF workflow | phi4:10t | 8600 chars | 11-15 min | >25 min = investigate |
| File write (simple) | phi4:14B | 3K chars | 20-30s | >60s = check context size |

### Resource Usage by Agent/Model

| Agent | Model | RSS | CPU (Idle) | CPU (Active) | Threads |
|-------|-------|-----|------------|--------------|---------|
| <AGENT3> | phi4:10t (14B) | 9-10 GB | 0-2% | 600-700% | 10 |
| <AGENT4> | qwen2.5-coder:1.5b | 1.0-1.5 GB | 0-1% | 80-150% | 8 |
| <AGENT5> | deepseek-r1:1.5b | 1.0-1.5 GB | 0-1% | 80-150% | 8 |
| <AGENT1> | qwen2.5:1.5b | 1.0-1.5 GB | 0-1% | 80-150% | 8 |
| <AGENT2> | qwen2.5:3b | 2.0-2.5 GB | 0-1% | 150-250% | 8 |

**Note:** CPU % = percentage of total CPU capacity (100% = 1 core, 800% = 8 cores)

### Acceptable Ranges

**CPU:**
- Idle: 5-20% (OS + background tasks)
- Single agent active: 50-80% (8-12 cores)
- Thermal throttling starts: >90°C sustained

**Memory:**
- Node2 (64GB): Safe up to 55GB, warning at 58GB, critical at 60GB
- Node6 (32GB): Safe up to 28GB, warning at 30GB, critical at 31GB
- Swap usage: <1GB acceptable, >2GB = performance problem

**Disk I/O:**
- SSD read/write: 200-500 MB/s normal, >800 MB/s = heavy load
- HDD read/write: 40-80 MB/s normal, >100 MB/s = saturated

**Load Average:**
- <core count = good (cores available for new work)
- =core count = fully utilized (no headroom)
- >core count = queue buildup (tasks waiting for CPU)

---

## Troubleshooting Guide

### Symptom: Agent Response Very Slow (>5 min for simple query)

**Diagnostic steps:**

1. **Check CPU utilization:**
   ```
   llm-monitor → CPU panel
   - If <50%: Not CPU-bound, check next
   - If >80% sustained: Normal for 14B model, check inference time baseline
   - If fluctuating 20-80%: Possible I/O wait or swap thrashing
   ```

2. **Check memory/swap:**
   ```
   llm-monitor → Memory panel
   - Swap used >2GB? → Root cause: swap thrashing
   - Solution: Kill unused models, reduce model size, or add RAM
   ```

3. **Check thermal throttling:**
   ```
   llm-monitor → CPU panel → CPU Temp
   - Temp >90°C? → Root cause: thermal throttling
   - Check: CPU Freq dropped below base clock?
   - Solution: Reduce thread count, improve cooling, or wait for cooldown
   ```

4. **Check context size:**
   ```
   Agent logs: Look for "context size: XXXXX chars"
   - >10K chars? → Root cause: context too large for CPU inference
   - Solution: Implement history truncation (800 chars per assistant response)
   ```

5. **Check model size mismatch:**
   ```
   curl http://localhost:11434/api/ps
   - Using 14B model for simple task? → Root cause: over-specification
   - Solution: Use 1.5B model for simple queries, 14B for complex reasoning
   ```

### Symptom: Agent Not Responding / Stuck

**Diagnostic steps:**

1. **Check queue status:**
   ```bash
   curl http://<NODE2_IP>:6000/status | jq
   # Check: queue_pending > 0?
   # Check: Which agent is in agents dict?
   ```

2. **Check Ollama process:**
   ```
   llm-monitor → Ollama panel
   - Status: active (green) = inference running, wait
   - Status: loaded (cyan) = model warm but idle, check agent
   - Model name shows which agent: phi4:10t (<AGENT3>) = <AGENT3> is owner
   ```

3. **Check agent process:**
   ```
   llm-monitor → Other AI Agents panel
   - Agent process exists? CPU %?
   - If CPU 0% and queue empty: Agent may be crashed
   - Check: tmux attach -t agent (see logs)
   ```

4. **Check dispatcher cooldown:**
   ```
   Agent just finished task? Wait 10s (Node2) or 30s (Node1) for cooldown
   Dispatcher status shows cooldown_sec value
   ```

### Symptom: "Out of Memory" or Agent Killed

**Root cause:** OOM killer terminated process due to RAM exhaustion

**Prevention:**
1. **Calculate RAM budget BEFORE loading models:**
   ```
   Available: 64GB
   OS + apps: -8GB
   Agent processes: -2GB
   Buffer: -4GB
   ──────────────
   Model budget: 50GB

   Model plan:
   - phi4:14B: 9GB
   - qwen2.5-coder:1.5b: 1GB
   - deepseek-r1:1.5b: 1GB
   - Reserve: 39GB (plenty of headroom)
   ```

2. **Monitor before hitting limit:**
   ```
   llm-monitor → Memory panel
   - Set mental threshold: Node2 = 55GB warning, 60GB critical
   - When approaching: Unload unused models with `ollama stop <model>`
   ```

3. **Design model swapping strategy:**
   ```python
   # Don't keep all models loaded 24/7
   # Load on demand, unload after use (for large models)

   def agent_task():
       ollama.load("phi4:10t")  # 9GB
       result = process_data()
       ollama.stop("phi4:10t")   # Free 9GB
       return result
   ```

### Symptom: High Swap Usage (>2GB)

**Impact:** 10-100x slower inference, disk thrashing, system feels frozen

**Diagnostic:**
```
llm-monitor → Memory panel
- Swap Used: 4.5 GB (red warning)
- RAM Used: 63.8 GB / 64 GB
- Swap Usage bar: Red (critical)
```

**Root cause:** Too many models loaded simultaneously

**Solution:**
1. **Immediate:** Unload models to free RAM
   ```bash
   # Check loaded models
   curl http://localhost:11434/api/ps

   # Unload unused models
   ollama stop <model_name>
   ```

2. **Long-term:** Implement model lifecycle management
   - Load models on demand
   - Set TTL for idle models (auto-unload after 30 min idle)
   - Use smaller models for non-critical tasks

3. **Hardware:** Consider RAM upgrade or workload distribution
   - Node2: Already maxed at 64GB (not upgradeable on 2019 MBP)
   - Solution: Distribute agents across Node6 (32GB) + Node2 (64GB)

### Symptom: Inference Slower Than Expected

**Example:** "Simple query takes 2 min with 1.5B model (should be 5-10s)"

**Diagnostic checklist:**

1. **Is another model already active?**
   ```
   llm-monitor → Ollama panel
   - Two models with "active" status? Ollama runs one at a time
   - Second model is queued, waiting for first to finish
   ```

2. **Is CPU throttled?**
   ```
   llm-monitor → CPU panel
   - Freq <base clock? (e.g., 1.8 GHz vs 2.4 GHz base)
   - Temp >90°C? Thermal throttling active
   ```

3. **Is system swapping?**
   ```
   llm-monitor → Memory panel → Swap
   - Any swap usage during inference = massive slowdown
   ```

4. **Is disk I/O saturated?**
   ```
   llm-monitor → System panel → Disk Read/Write
   - SSD >1000 MB/s or HDD >120 MB/s = saturated
   - Possible model loading from disk (not cached in RAM)
   ```

5. **Check actual thread count:**
   ```bash
   # Verify model is using correct thread count
   ps aux | grep ollama_llama_server
   # Count CPU % (e.g., 650% = ~6-7 threads, not 10 as configured)

   # If mismatch, recreate model with explicit thread parameter:
   ollama create model:Nt -f Modelfile  # N = thread count
   ```

---

## Alerts & Thresholds

### Critical Alerts (Immediate Action Required)

| Metric | Threshold | Action |
|--------|-----------|--------|
| RAM usage | >95% or swap >5GB | Kill non-essential processes, unload models |
| CPU temp | >100°C sustained | Reduce thread count or stop inference, check cooling |
| Disk space | <5GB free | Clean logs, move data to Storage1, expand partition |
| Agent crash | Process not found | Check logs, restart agent, investigate root cause |
| OOM kill | dmesg shows OOM | Review RAM budget, reduce model count/size |

### Warning Alerts (Monitor Closely)

| Metric | Threshold | Action |
|--------|-----------|--------|
| RAM usage | >85% or swap >2GB | Plan to unload models, avoid loading additional |
| CPU temp | >90°C sustained | Consider reducing thread count or adding cooldown |
| Load average | >1.5× core count | Review dispatcher queue, check for stuck tasks |
| Disk space | <20GB free | Schedule cleanup, review backup retention |
| Inference time | 2× baseline | Check CPU throttling, memory pressure, context size |

### Info Alerts (Awareness)

| Metric | Threshold | Action |
|--------|-----------|--------|
| RAM usage | >70% | Normal for multi-model operation, monitor trend |
| CPU usage | >80% sustained | Normal during inference, check completion time |
| Queue depth | >2 tasks | Normal during busy period, monitor for growth |
| Model loaded | New model detected | Verify expected, check RAM impact |

---

## Integration with Agent-Central

### State File Location

**Path:** `~/agent-central/state/ollama/agents.json`

**Purpose:** Maps Ollama models → agent names for monitoring visibility

**Format:**
```json
{
  "agents": {
    "phi4:10t": {
      "agent": "<AGENT3>",
      "pid": 12345,
      "registered_at": "2026-03-30T14:23:15"
    },
    "qwen2.5-coder:1.5b": {
      "agent": "<AGENT4>",
      "pid": 12346,
      "registered_at": "2026-03-30T14:23:20"
    },
    "deepseek-r1:1.5b": {
      "agent": "<AGENT5>",
      "pid": 12347,
      "registered_at": "2026-03-30T14:23:25"
    }
  }
}
```

**How llm-monitor uses this:**
1. Reads state file at startup and each refresh
2. Maps model names to agent names
3. Displays in Ollama panel: `phi4:10t (<AGENT3>)` with status and metrics
4. Shows in Other AI Agents panel: agent process with model name

**Manual verification:**
```bash
# Check what agent is using what model
cat ~/agent-central/state/ollama/agents.json | jq

# Cross-reference with loaded models
curl http://localhost:11434/api/ps | jq

# Should match: model in agents.json = model in /api/ps
```

### Agent Registration Flow

1. **Agent starts** → Calls Ollama API with specific model
2. **Agent-central dispatcher** → Detects model load via `/api/ps` polling
3. **State file updated** → Maps model → agent name with PID and timestamp
4. **llm-monitor reads** → Shows `model (agent)` in UI

**Benefits:**
- No manual configuration in llm-monitor
- Automatic agent discovery
- Accurate agent → model → resource mapping
- Survives monitor restarts (state persists in file)

---

## Best Practices Summary

### Daily Operations

1. **Start your session:**
   ```bash
   # On Node2 (primary)
   cd ~/llm-monitor
   sudo ./monitor

   # Let it run in a dedicated terminal or tmux pane
   # Check it periodically during heavy agent usage
   ```

2. **Before starting heavy workload:**
   - Check current RAM usage
   - Verify no swap usage
   - Check CPU temp is baseline (<60°C)
   - Confirm no other models actively inferring

3. **During long-running tasks:**
   - Monitor CPU % and temp every 5-10 min
   - Watch for thermal throttling (temp >90°C)
   - Check inference isn't stuck (CPU should be active)

4. **After task completion:**
   - Check RAM freed (model still loaded but not actively using CPU)
   - Verify swap returned to 0 or minimal
   - Review inference time vs baseline (was it normal?)

### Capacity Planning

1. **Model budget per machine:**
   ```
   Node2 (64GB): Up to 50GB in models (prefer 30-40GB for safety)
   Node6 (32GB): Up to 24GB in models (prefer 18-22GB for safety)
   Node3 (16GB): Up to 12GB in models (prefer 8-10GB for safety)
   Node4 (16GB): Up to 12GB in models (prefer 8-10GB for safety)
   Node5 (12GB): Up to 8GB in models (prefer 4-6GB for safety)
   ```

2. **When to distribute workload:**
   - Single machine RAM >80%: Move agents to other nodes
   - Queue depth >3 sustained: Add second dispatcher on Node6
   - Thermal throttling: Use cooler machine (M1 Max) for sustained load

3. **When to upgrade hardware:**
   - Swap usage >2GB regularly: Need more RAM (but Node2 maxed out)
   - Queue depth >5 regularly: Need more CPU cores or second machine
   - Storage <20GB free: Need larger SSD or offload to Storage1

### Automation Opportunities

**Future enhancements:**

1. **Automated alerts:**
   ```python
   # Send Telegram alert when:
   # - RAM >90% or swap >3GB
   # - CPU temp >95°C sustained >5 min
   # - Agent not responding >10 min
   # - Queue depth >5 sustained >15 min
   ```

2. **Health check endpoint:**
   ```bash
   # HTTP endpoint returning fleet health
   curl http://<NODE2_IP>:9999/health
   {
     "node2": {"status": "healthy", "ram_pct": 72, "temp": 78},
     "node6": {"status": "warning", "ram_pct": 88, "temp": 65}
   }
   ```

3. **Automatic model unloading:**
   ```python
   # Unload models idle >30 min when RAM >80%
   # Agent loads on-demand when task submitted
   ```

4. **Load balancing:**
   ```python
   # Route new tasks to least-loaded machine
   # Prefer Node2 (most RAM) but use Node6 if Node2 >80% RAM
   ```

---

## Conclusion

**Monitoring is not optional** for local LLM infrastructure. Without visibility:
- Resource exhaustion causes silent failures
- Thermal issues create mysterious slowdowns
- Queue congestion appears as "agent not responding"
- Swap thrashing makes inference 10-100x slower

**With proper monitoring (llm-monitor + API endpoints):**
- Catch problems before failure (RAM >85% → unload models now)
- Optimize performance (tune thread count based on temp/speed data)
- Understand agent behavior (see exactly which agent is using which model)
- Plan capacity (know when to distribute workload or upgrade hardware)

**Key principle:** Metrics drive decisions. Don't guess, measure.

---

**See Also:**
- `2FLEETS.md` - Hardware specifications and capabilities
- `2APIHUB.md` - Agent API endpoints and dispatcher architecture
- `2AGENT_ARCHITECTURE.md` - Lessons learned, performance baselines, known issues
- `~/llm-monitor/README.md` - Monitor installation and usage details

**Maintenance:** Update baselines after hardware changes, OS upgrades, or model changes. Monitor should reflect reality, not outdated assumptions.
