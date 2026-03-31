# Multi-Agent AI Infrastructure Documentation

> **Production-grade documentation for a distributed local LLM infrastructure running multiple specialized AI agents with comprehensive monitoring and resource management.**

[![Platform](https://img.shields.io/badge/platform-macOS%20%7C%20Linux-blue)]()
[![Python](https://img.shields.io/badge/python-3.8%2B-blue)]()
[![LLM](https://img.shields.io/badge/LLM-Ollama%20%7C%20Local-green)]()

---

## Overview

This repository contains **production-grade infrastructure documentation** for a distributed multi-agent AI system running entirely on local hardware. The infrastructure supports multiple specialized AI agents performing different tasks (health tracking, financial operations, document extraction, email automation, chat) with intelligent resource management, monitoring, and optimization.

**Key Achievements:**
- ✅ **5-node compute fleet** with 152GB total RAM orchestrating 5+ specialized AI agents
- ✅ **Real-time monitoring system** tracking CPU, GPU, memory, thermals, and agent-specific metrics
- ✅ **Multi-model optimization** running 1.5B to 14B parameter models with intelligent resource allocation
- ✅ **Production reliability** with queue management, priority scheduling, and thermal throttling prevention
- ✅ **Comprehensive observability** correlating system metrics with agent behavior for performance tuning

---

## Repository Structure

### 📊 Core Documentation

| Document | Purpose | Key Highlights |
|----------|---------|----------------|
| **[MONITORING.md](MONITORING.md)** | Monitoring strategy & best practices | Why monitoring is essential, performance baselines, troubleshooting guide, 24/7 ops considerations |
| **[2FLEETS.md](2FLEETS.md)** | Hardware inventory & capabilities | 5 compute nodes (152GB RAM), 2 mobile devices, storage architecture, deployment recommendations |
| **[2APIHUB.md](2APIHUB.md)** | API registry & service architecture | Agent APIs, authentication, dispatcher pattern, cross-agent handoffs, Ollama integration |
| **[2AGENT_ARCHITECTURE.md](2AGENT_ARCHITECTURE.md)** | Lessons learned & architecture patterns | What works/doesn't work, model performance, PDF processing pipelines, context management |

### 🎯 Target Audience

This documentation is designed for:
- **Infrastructure Engineers** managing AI/ML workloads
- **MLOps/LLMOps Professionals** optimizing local inference
- **System Architects** designing distributed agent systems
- **Technical Recruiters** at AI companies (NVIDIA, Anthropic, OpenAI, etc.)

---

## Infrastructure Highlights

### Multi-Agent System Architecture

**5 Specialized Agents Running Concurrently:**
- `<AGENT1>` - General chat & personal assistant (qwen2.5:1.5b, priority 1)
- `<AGENT2>` - Email campaigns & automation (qwen2.5:3b, priority 0)
- `<AGENT3>` - Health data tracking & analysis (phi4:14B, priority 1)
- `<AGENT4>` - Financial operations & bookkeeping (qwen2.5-coder:1.5b, priority 0)
- `<AGENT5>` - Document extraction service (deepseek-r1:1.5b, stateless)

**Key Design Patterns:**
- Priority-based dispatcher (business-critical agents process first)
- Stateless service pattern for document extraction (bypasses queue)
- Cross-agent handoffs for specialized tasks
- Model-to-agent mapping via state file for monitoring visibility

### Compute Fleet

| Node | Hardware | RAM | Role | Key Capabilities |
|------|----------|-----|------|------------------|
| **Node2** ⭐ | Intel i9-9980HK, AMD GPU | 64GB | Primary orchestrator | Main AI brain, 16 threads, heaviest workloads |
| **Node6** | Apple M1 Max, 24-core GPU | 32GB | Mobile heavy node | Best GPU (Metal 4), ANE, unified memory |
| **Node3** | Intel i7 Quad-Core | 16GB | Security/utility | Log aggregation, backups, monitoring |
| **Node4** | Intel i7 Dual-Core | 16GB | Edge worker | Arch Linux, Tailscale mesh, lightweight tasks |
| **Node5** | Intel i7 Dual-Core | 12GB | Storage node | 1TB HDD, bulk archival, backup coordination |

**Total Resources:**
- **152GB RAM** across fleet (140GB desktop + 12GB mobile)
- **2.55TB SSD + 1TB HDD** local storage
- **2TB USB** shared infrastructure storage
- **7 total nodes** (5 desktop/laptop + 2 mobile)

### Real-Time Monitoring System

**[llm-monitor](https://github.com/anthropics/llm-monitor)** - Custom terminal dashboard providing:

**System Metrics:**
- CPU: Per-core utilization, frequency, P/E core split, load average, temps
- GPU: Utilization, frequency, power draw (macOS via powermetrics, Linux/Windows via nvidia-smi)
- ANE: Apple Neural Engine power usage (Apple Silicon)
- Memory: Unified memory breakdown (wired, compressed, cached, buffers)
- Thermal: CPU/GPU die temps with throttling detection
- Power: Package/CPU/GPU/ANE power draw (macOS, Linux RAPL)

**AI Process Tracking:**
- **Ollama processes:** Model name, CPU%, RSS, inference state (active/loaded/idle)
- **Coding agents:** Claude Code + Aider with model detection via config files
- **Custom agents:** Multi-agent orchestrator with model-to-agent mapping
- **Queue visibility:** Dispatcher queue depth, cooldown timers, priority handling

**Agent-Central Integration:**
- Reads state file to map models → agent names
- Example display: `phi4:10t (<AGENT3>)` shows which agent is using which model
- Automatic discovery, no manual configuration

---

## Key Technical Achievements

### 1. Performance Optimization

**Thread Count Tuning:**
- Ollama thread count directly impacts inference speed vs thermal load
- Created custom models with explicit thread parameters (e.g., `phi4:10t` = 10 threads)
- Benchmarked 8/10/12/16 thread configs on real workloads
- Result: 10 threads = 62% CPU = +25% faster inference without thermal throttling

**Context Management:**
- Discovered 14B models timeout with >10K char context on CPU
- Implemented history truncation (800 chars per assistant response)
- Reduced inference time from 15+ min (timeout) to 2-5 min (acceptable)
- Skip file injection when history >5K chars (prevents context explosion)

**Adaptive PDF Processing:**
- Multi-layer fallback: PyMuPDF → PyPDF2 → OCR
- Adaptive chunking: 2500 chars for 14B (CPU inference time), 900 chars for 1.5B
- Dedicated extraction service (<AGENT5>) using 1.5B model for speed
- Architecture: Fast extraction (4 min) + quality formatting (agent's model) = 30+ min savings

### 2. Resource Management

**RAM Budget Planning:**
```
Node2 (64GB):
  OS + apps:           8GB
  Agent processes:     2GB
  Model budget:       50GB  (safe operating envelope)
  Buffer:             4GB   (emergency headroom)

Typical load:
  - phi4:14B:          9GB
  - qwen2.5:1.5b:      1GB
  - deepseek-r1:1.5b:  1GB
  = 11GB models + 8GB overhead = 19GB total (plenty of headroom)
```

**Swap Thrashing Prevention:**
- Monitor swap usage: >2GB = critical (10-100x slower inference)
- Unload idle models when RAM >80%
- Size models to available RAM (prefer 3×1.5B over 3×14B on constrained nodes)

### 3. Monitoring-Driven Decisions

**Thermal Throttling Detection:**
```
Symptom: 15-minute timeout on task that should take 10 minutes
Monitoring showed:
  T+0:    Task start, CPU 100%, 60°C ✓
  T+8min: CPU 100%, 95°C (warning)
  T+10min: CPU freq drops 2.4GHz → 1.8GHz (throttling!)
  T+15min: Timeout (inference 40% slower)
Root cause: Sustained 100% CPU → thermal throttling
Solution: Reduce thread count 16→10 OR improve cooling
```

**Queue Congestion Visibility:**
```bash
# Without monitoring: "Agent not responding"
# With monitoring:
curl http://<NODE2_IP>:6000/status
{
  "queue_pending": 2,  # ← 2 tasks waiting
  "agents": {
    "<AGENT4>": {"alive": true}  # ← Priority 0 agent running
  }
}
# Insight: <AGENT3> is queued behind <AGENT4> (expected, by design)
```

### 4. Production-Grade Architecture

**Dispatcher Pattern:**
- Single shared queue per machine (one job at a time)
- Priority 0 agents (<AGENT4>, <AGENT2>) = business-critical, process first
- Priority 1 agents (<AGENT3>, <AGENT1>) = background/personal tasks
- Configurable cooldown between tasks (10s Node2, 30s Node1)

**Stateless Service Pattern:**
- <AGENT5> document extraction bypasses dispatcher entirely
- Processes requests immediately (no queue)
- Uses requesting agent's bot token for multi-tenancy
- Returns structured JSON for agent-specific formatting

**Agent-Central State Management:**
- Persistent state file maps models → agents → PIDs
- Enables monitoring without configuration
- Survives restarts (state persists)
- Automatic agent discovery via Ollama API polling

---

## Technical Stack

**Infrastructure:**
- **Hardware:** 5 macOS/Linux nodes (Intel + Apple Silicon + AMD GPUs)
- **OS:** macOS 26.3.1, Arch Linux, Xubuntu 20.04
- **Orchestration:** Custom agent-central dispatcher (Python)
- **Monitoring:** llm-monitor (Python, psutil, rich)

**AI/ML:**
- **LLM Serving:** Ollama (local inference)
- **Models:** phi4:14B, qwen2.5-coder (1.5B/7B), deepseek-r1:1.5b, qwen2.5:3b
- **Framework:** Custom agent framework with tool calling, memory management
- **Integration:** Telegram bots, Gmail API, HTTP APIs

**Observability:**
- **System Metrics:** CPU, GPU, ANE, memory, thermals, power (macOS powermetrics, Linux RAPL)
- **Process Tracking:** Ollama runners, agent processes, model-to-agent mapping
- **API Monitoring:** Flask/FastAPI health endpoints, queue depth, agent state

---

## Key Learnings & Best Practices

### What Works ✅

1. **phi4:14B for tool calling** - Reliable parameter names, follows instructions, quality output
2. **History truncation** - Prevents context explosion (800 chars per assistant response)
3. **Adaptive chunking** - 2500 chars for 14B CPU inference, 900 chars for 1.5B models
4. **PyMuPDF primary extraction** - Most robust for PDFs, fallback to PyPDF2 → OCR
5. **Thread count tuning** - 10 threads = sweet spot (62% CPU, good thermal margin)
6. **Tool-first prompts** - Put tool instructions at top of system prompt (model pays attention)
7. **Priority dispatching** - Business-critical agents (priority 0) process first
8. **Monitoring-driven optimization** - Metrics reveal root causes (thermal, swap, context size)

### What Doesn't Work ❌

1. **qwen2.5-coder:7B for tools** - Unreliable parameter names, hallucinates success
2. **Full responses in history** - 2K+ char responses → 15+ min inference or timeout
3. **Always injecting files** - Ignore context size → unnecessary data for simple writes
4. **Timeout without chunking** - 14B + 10K chars context = CPU inference timeout
5. **JSON without preprocessing** - Models add control chars (\n, \r) that break parsing
6. **Assuming thread count applies** - Must verify via custom Modelfile (API param unreliable)

### Performance Baselines

| Task | Model | Context | Time | Warning Threshold |
|------|-------|---------|------|-------------------|
| Simple chat | phi4:14B | 2K | 20-40s | >60s |
| Simple chat | qwen2.5:1.5b | 2K | 3-8s | >15s |
| PDF extraction | deepseek-r1:1.5b | 5500 | 4 min | >8 min |
| PDF processing | phi4:10t | 7500 | 10-12 min | >20 min |
| Full workflow | phi4:10t | 8600 | 11-15 min | >25 min |

---

## Why This Matters

### For NVIDIA
- **GPU Optimization:** ANE/Metal monitoring, VRAM tracking, GPU utilization analysis
- **Local Inference:** Real-world performance data for CPU vs GPU inference trade-offs
- **Thermal Management:** Sustained workload optimization, throttling detection, power monitoring
- **Multi-Model Serving:** Resource allocation strategies for concurrent model execution

### For Anthropic
- **LLM Operations:** Production deployment of multiple Claude-class models (14B params)
- **Context Management:** Real-world learnings on context size vs inference time
- **Tool Calling:** Reliability analysis across different model sizes and architectures
- **Monitoring & Observability:** Agent behavior correlation with system metrics
- **Multi-Agent Systems:** Priority scheduling, dispatcher patterns, cross-agent handoffs

### For MLOps/LLMOps Roles
- **Resource Management:** RAM budgeting, model sizing, swap prevention
- **Performance Optimization:** Thread tuning, context optimization, thermal awareness
- **Production Reliability:** Monitoring, alerting thresholds, troubleshooting methodology
- **Capacity Planning:** When to scale horizontally, vertical limits, workload distribution

---

## Documentation Quality

This documentation demonstrates:

✅ **Systems Thinking** - Hardware → OS → Runtime → Application correlations
✅ **Data-Driven Decisions** - Every optimization backed by measurements
✅ **Production Mindset** - Monitoring, alerting, capacity planning, troubleshooting
✅ **Technical Depth** - CPU throttling, swap thrashing, context explosion, tool calling reliability
✅ **Clear Communication** - Complex technical concepts explained with examples and baselines
✅ **Operational Excellence** - 24/7 considerations, thermal awareness, resource budgeting

---

## Future Roadmap

**Planned Enhancements:**
- [ ] GPU acceleration for inference (M1 Max 24-core GPU, AMD Radeon Pro 5500M)
- [ ] Automated alerting via Telegram (RAM >90%, temp >95°C, queue depth >5)
- [ ] Load balancing across fleet (route to least-loaded node)
- [ ] Model lifecycle management (auto-unload idle models when RAM >80%)
- [ ] Health check HTTP endpoint (fleet-wide status monitoring)
- [ ] Prometheus/Grafana integration (long-term metrics & dashboards)
- [ ] Docker Swarm / Kubernetes cluster (container orchestration)
- [ ] Centralized logging with ELK stack (log aggregation & search)

---

## Contact

This infrastructure documentation showcases real-world AI/ML operations expertise. If you're interested in discussing:
- AI infrastructure architecture
- Local LLM optimization
- Multi-agent systems design
- MLOps/LLMOps best practices

Feel free to open an issue or reach out.

---

## License

Documentation is provided as-is for educational and portfolio purposes.

---

**Last Updated:** 2026-03-31
**Maintained by:** Infrastructure Engineer specializing in AI/ML Systems
