# DOMOVOY SETUP — AI Agent on Any Machine

How to configure an AI coding/sysadmin agent (the "domovoy") on a Linux machine. This is a companion to SYSTEM_SETUP.md — SYSTEM_SETUP builds the operating system, this document configures the AI agent layer on top.

**Audience:** Anyone deploying an AI domovoy on their own hardware. Written to be shared — no machine-specific secrets, no internal paths. Our concrete setup (an Intel Arc GPU machine running Arch Linux) appears as worked examples throughout.

---

## Directory conventions

```
  /home/domovoy/
    Public/                   Canonical source repos (Domovoy's own clones)
      domovoy-bootstrap/       Identity, templates, bootstrap/migration docs
      domovoy-skills/          Skill library ("farmer's market" = local clone of the store)
    .agents/                   Agent identity & runtime skills
      AGENTS.md                Core instructions (loaded at session start)
      skills/                  "Fridge" — Syncthing-synced across fleet
    .config/opencode/          opencode server config
    .local/                    opencode session state, database, health-check script
    setup/<hostname>/           Per-machine profiles (SYSTEM_INFO, SYSTEM_SETUP, ssh.pub)
    maintenance/reports/        Session binlog
    models/                     GGUF models for local llama.cpp inference
    .ssh/                       SSH keys (id_domovoy — git/GitHub + fleet access)
```

The **store** (`github.com/alexindigo/domovoy-skills`) is the canonical public
skill library. Skills flow: store → `~/Public/domovoy-skills/` (local clone) →
`~/.agents/skills/` (fridge). Each Domovoy maintains its own `~/Public/` clones
and commits with a fingerprint-derived email (`<8-hex-chars>@domovoy`).

---

## 1. Identity & Conventions

The domovoy runs as a dedicated system user (UID 588) with passwordless sudo.
OS-level setup (user creation, `sudoers`, `loginctl enable-linger`) is covered
in **BOOTSTRAP.md** (same repo). That document is the step-by-step D-Day
operation — paste it into a root opencode and follow along.

Once the domovoy user exists with SSH keys on GitHub and both repos cloned to
`~/Public/`, come back here for the AI agent layer.

### AGENTS.md — the identity file

The minimum viable AGENTS.md:

```markdown
# Assistant Instructions

You are the **domovoy** — a personal tech support and sysadmin agent.
Run as the `domovoy` user. Use `sudo` for elevated operations.

## Safety rules
- Read-only commands are allowed freely.
- Any command that writes, modifies, installs, or deletes requires explicit approval.
- NEVER open ports or enable network services without approval.
- NEVER restart the opencode server — it kills your own connection.

## Maintenance report
Load the `maintenance-report` skill at the start of every session.
It defines the daily binlog at `maintenance/reports/`.

## Unexpected situations
STOP immediately. Explain what happened. Let the user decide.
```

The file will grow as you add conventions, but this is enough to start safely.

---

## 2. opencode Service Configuration

The opencode user service, health-check watchdog, nightly restart timer, and
systemd unit files are covered in **BOOTSTRAP.md**. That document contains the
step-by-step commands to create, enable, and start all services. It also covers
network access (nftables) and Syncthing setup.

### Critical rule: NEVER restart opencode yourself

The domovoy runs INSIDE the opencode server. Restarting it kills the connection mid-session. When a change requires a server restart, create a script and ask the human user to run it.

---

## 3. AI Model Provider — The Decision Framework

This is the most consequential choice. The domovoy needs a model to think with. You have two paths: remote API or local model.

### 3.1 Remote API vs Local Model

| Factor | Remote API | Local Model |
|--------|-----------|-------------|
| **Setup** | API key + one config line | GPU driver + model download + server |
| **Ongoing cost** | Per-token billing | Electricity only |
| **Privacy** | Code sent to third party | Everything stays on your machine |
| **Internet** | Required | Not required (offline capable) |
| **Model quality** | Best available (frontier models) | Good, but smaller than cloud giants |
| **Latency** | Network round-trip | Local (GPU-dependent) |
| **GPU required** | No | Yes (unless CPU-only) |

**Remote API example (opencode.jsonc):**
```jsonc
{
    "model": {
        "provider": "anthropic",
        "name": "claude-sonnet-4-20250514"
    }
}
```

**Local model example (opencode.jsonc):**
```jsonc
{
    "model": {
        "provider": "openai-compatible",
        "name": "qwen3.5-14b",
        "base_url": "http://localhost:8080/v1"
    }
}
```

openconde supports any OpenAI-compatible endpoint — llama.cpp, Ollama, vLLM, and most local runners qualify.

If you choose remote API, stop here — the rest of this section is about running models locally.

---

### 3.2 VRAM → Model Size

Local models come in "quantized" formats. Quantization reduces precision to fit models in less memory. The trade is a small quality loss for a large size reduction.

**Common quantization formats (GGUF):**

| Format | Bits per weight | VRAM per 1B params | 7B model size | 14B model size | Quality |
|--------|----------------|-------------------|--------------|---------------|---------|
| Q4_K_M | ~4.5 | ~0.7 GB | ~5 GB | ~10 GB | Good (sweet spot) |
| Q5_K_M | ~5.5 | ~0.85 GB | ~6 GB | ~12 GB | Slightly better |
| Q8_0 | ~8.5 | ~1.05 GB | ~8 GB | ~15 GB | Near-perfect |
| FP16 | 16 | ~2.1 GB | ~15 GB | ~30 GB | Reference |

Q4_K_M is the recommended starting point — it preserves most of the model's quality at roughly half the VRAM of FP16.

**Add ~2–8 GB for context (KV cache).** A 96K context window on a 14B model adds ~6 GB.

**Practical GPU sizing:**

| GPU VRAM | Fits at Q4_K_M | Fits at Q8_0 | Example cards |
|----------|---------------|-------------|---------------|
| 4 GB | 1–3B | — | Older / iGPU |
| 6–8 GB | 7B comfortably, 14B tight | 7B tight | RTX 2060/3060/4060, Arc A750, Arc B50 |
| 12 GB | 14B comfortably | 7B + context | RTX 4070/5070, Arc B580, RX 7700 XT |
| 16 GB | 14B + context | 14B tight | RTX 4080/5080, RX 7900 XT |
| 24 GB | 32B | 14B + context | RTX 4090/5090 |

**Example — for an Intel Arc GPU (e.g., Arc B50) (~8–12 GB VRAM):** 7–14B models at Q4_K_M. A 7B model fits easily (~5 GB), a 14B model is comfortable (~10 GB). 32B models would exceed available VRAM.

**If you have NO GPU:** CPU inference works but is slower. llama.cpp runs on CPU. Expect ~10–25 tok/s on a modern 16-core CPU for a 7B model (about one-third to one-half the speed of a mid-range GPU). The same VRAM sizing rules apply — the model still needs to fit in system RAM.

---

### 3.3 Which Model Families Suit Sysadmin Work?

A good model for computer maintenance needs to handle: shell commands, config file editing, scripting (Bash, Python), systemd unit files, package management, debugging error messages, and understanding system architecture.

As of mid-2026, these families lead:

#### Tier 1 — Strongest (need 12+ GB VRAM for best sizes)

| Family | Best size | Why |
|--------|----------|-----|
| **Qwen3.6** (Alibaba, June 2026) | 27B | Latest release; targets "productive coding experience"; strong across shell, configs, and general sysadmin |
| **DeepSeek-R1-Distill** | 14B, 32B | Chain-of-thought reasoning excels at debugging — understands subtle error messages and traces root causes |
| **Qwen3-Coder** | 30B MoE | Coding-specific with 256K context; handles large config files and multi-step edits |
| **Gemma 4** (Google, May 2026) | 12B, 26B | Strong general reasoning; 12B fits on modest GPUs |
| **Devstral 2** (Mistral) | 24B | Agentic coding; designed for tool use across files |
| **Codestral** (Mistral) | 22B | 80+ languages; good code completion and instruction following |

#### Tier 2 — Lightweight (fit on 6–12 GB VRAM)

| Family | Best size | Why |
|--------|----------|-----|
| **Qwen3.5** | 7B, 14B | Solid all-rounder; 14B is the sweet spot for most sysadmin tasks |
| **Granite 4.1** (IBM) | 8B | Strong tool calling at small size — good for shell commands and API scripts |
| **Ministral 3** (Mistral) | 8B, 14B | Efficient small models; "best of class cost-to-performance" |
| **Phi-4** (Microsoft) | 14B | Strong reasoning at moderate size |

#### Tier 3 — Ultra-light (fit on 2–4 GB, CPU or iGPU)

| Family | Best size | Why |
|--------|----------|-----|
| **Qwen3.5** | 2B, 4B | Runs on anything; basic scripting, simple commands |
| **Granite 4.1** | 3B | Tool calling at tiny size |
| **Phi-4** | 3B | Good reasoning in a very small package |

**Minimum for real sysadmin work:** 7B. Below that, models can handle simple one-liners but fail at multi-step debugging or understanding novel errors.

**Sweet spot:** 14B at Q4_K_M (~10 GB VRAM). Handles 90%+ of real administration tasks — systemd units, nftables rules, btrfs commands, shell scripting, config file refactoring.

**When to go bigger (32B+):** Only if you regularly need deep multi-file refactoring, complex security audits, or reasoning about subtle race conditions. For daily sysadmin, 14B is usually the better choice because it's faster and leaves VRAM for context.

#### What about switching between models?

The decision framework above helps you pick a starting model, but a single setup can run several models and switch between them depending on the task. For now, pick one model as the daily driver and iterate. Rules for automated model switching per task type can be added later.

**Example — for an Intel Arc GPU (e.g., Arc B50) (16 GB GDDR6):**

| Model | Size | VRAM | Use case |
|-------|------|------|----------|
| Qwen2.5-14B Q4_K_M | 14B | ~9 GB | Daily driver — shell commands, configs, scripts |
| DeepSeek-R1-Distill-Qwen-14B Q4_K_M | 14B | ~10 GB | Debugging sessions (chain-of-thought helpful for tricky errors) |
| Granite-4.1-8B Q5_K_M or Q8_0 | 8B | ~7–8 GB | Fast tool-calling tasks, API scripts |

Start with the 14B daily driver. The 16 GB VRAM leaves ~6 GB for context — comfortable for 32K+ windows. Switch to reasoning model when debugging. The 8B is a backup for fast responses or when running multiple models simultaneously.

---

### 3.4 Runner Selection by GPU

The "runner" is the inference engine that loads the model and serves an API. Your GPU type determines which runner backends are available.

| GPU Vendor | Best Backend | Typical Package | Notes |
|-----------|-------------|----------------|-------|
| **NVIDIA** | CUDA (llama.cpp) | `cuda` toolkit + llama.cpp | Mature, fast, widely available |
| **NVIDIA** | vLLM | pip / Docker | Production-grade, higher throughput |
| **AMD** | ROCm (llama.cpp) | `rocm` + llama.cpp | Works but less polished than CUDA |
| **Intel Arc** | SYCL (llama.cpp) | oneAPI DPC++ + llama.cpp SYCL | Best performance on Intel GPUs; Battlemage verified |
| **Intel Arc** | Vulkan (llama.cpp) | `llama-cpp-vulkan` (no extra deps) | Works, but no XMX optimization — ~40% slower than SYCL |
| **Apple Silicon** | Metal (llama.cpp / MLX) | Built-in | Excellent performance; unified memory |
| **No GPU** | CPU (llama.cpp) | `llama.cpp` (no GPU backend) | Works everywhere; 2–5× slower than GPU |

All of these serve OpenAI-compatible APIs that opencode can use.

**Example — for an Intel Arc GPU (e.g., Arc B50) (Battlemage G21, Xe2 architecture):**

The SYCL backend provides the best performance on Intel Arc by leveraging the XMX matrix engines. The Vulkan backend is a lighter-weight alternative that works without the oneAPI toolkit (~ few GB install), but at reduced speed.

---

### 3.5 How to Get the Model Files

GGUF (GPT-Generated Unified Format) is the standard format for local models. Files are available from:

- **HuggingFace** — the primary source. Search for model name + "GGUF", e.g., `bartowski/Qwen3.5-14B-Instruct-GGUF`
- **LM Studio** — GUI catalog with one-click downloads
- **Ollama model library** — `ollama pull qwen3.5:14b` (if using Ollama as runner)

A 14B Q4_K_M GGUF file is typically ~8–10 GB. Download it once, store it wherever you keep model files.

---

### 3.6 Multi-Model Pool Architecture

A single model handles most tasks, but a pool of specialized models — each optimized for a different type of work — gives better results. A lightweight model parses data quickly; a reasoning model diagnoses subtle bugs; a heavy coding model handles multi-file refactors.

#### The Four Roles

| Role | Size range | What it does | Model family examples |
|------|-----------|-------------|----------------------|
| **Fast** ⚡ | 3–9B | Parse log files, fetch URLs, run shell one-liners, feed structured data to heavier models. Runs alongside Daily. | Granite (tool calling), Ornith (agentic coding), Qwen 3/3.5 small variants |
| **Daily** 📋 | 9–14B | Log analysis, report assembly, package update checks, AUR safety validation. The daily driver — handles 80% of sysadmin work. | Gemma 4 (modern, tool calling, 256K ctx), Qwen 2.5/3 Instruct (battle-tested) |
| **Think** 🧠 | 14–32B | Plan-mode investigation, root cause analysis, chain-of-thought debugging. Swapped in for hard problems. | DeepSeek-R1 distilled (reasoning), Qwen with thinking mode |
| **Coder** 🏋️ | 14–32B | Multi-file refactors, complex system configuration, novel scripting. Heaviest model, used sparingly. | Qwen3.6 (best local coding, SWE-bench 77.2%), DeepSeek-Coder, Ornith |

#### VRAM Budgeting

Determine your usable VRAM: GPU VRAM minus display/compositor overhead (~1–2 GB for a desktop GUI at 3440×1440, negligible for headless).

llama-server's router mode (since Dec 2025 — PR #17470) supports multiple models on one port. Specify `"model"` in the API request to route to the right model. The server auto-loads models on first request and auto-unloads idle ones (via `--sleep-idle-seconds`). No server restarts, no manual model management.

**What to keep loaded:**

| Always-on pair | Why |
|----------------|-----|
| Fast + Daily | Handles 80% of daily tasks — one parses data, one interprets |
| Think | Swap in when debugging/planning (auto-unload Daily or Fast) |
| Coder | Swap in for heavy code work (auto-unload everything) |

Both Fast and Daily must fit in usable VRAM simultaneously. Example:
- 14 GB usable → Fast (5.5 GB) + Daily (7.7 GB) = 13.2 GB ✅
- 8 GB usable → Fast only, swap Daily in/out
- 24 GB usable → Fast + Daily + Think all loaded ✅

#### How Switching Works

The llama-server router uses a multi-process architecture. The main process listens on your port (e.g. 8080) and spawns child processes for each model. When a request arrives:

1. Router reads the `"model"` field in the API request
2. If the model isn't loaded, it auto-loads (spawns a child process)
3. If VRAM is tight, it auto-unloads idle models (after `--sleep-idle-seconds`)
4. Request is forwarded to the correct model's child process

The `GET /v1/models` endpoint lists all models and their load status (`loaded`, `sleeping`, `unloaded`). POST `/models/load` and `/models/unload` give manual control if needed.

#### Configuring Per-Agent Models in opencode

open code supports different models per agent (Build, Plan, Explore). Map your model pool to opencode's agents:

```jsonc
{
  "model": "llama-daily/gemma-4-12b-it",         // default = Daily
  "small_model": "llama-fast/granite-4.1-8b",    // background tasks = Fast
  "agent": {
    "plan": { "model": "llama-think/deepseek-r1-14b" },
    "explore": { "model": "llama-fast/granite-4.1-8b" }
  }
}
```

- **Build** (default) → Daily driver — most sysadmin work
- **Plan** → Think model — strategic analysis, investigation
- **Explore** (subagent) → Fast model — codebase search, data gathering

#### Our Model Pool (Worked Example)

VRAM budget: Arc Pro B50 (16 GB), ~14 GB usable after GUI.

| Role | Model | Quant | Size | HuggingFace Download (curl) |
|------|-------|-------|------|----------------------------|
| **Fast ⚡** | **Granite-4.1-8B** *(primary)* | Q4_K_M | 5.0 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/granite-4.1-8b-Q4_K_M.gguf https://huggingface.co/ibm-granite/granite-4.1-8b-GGUF/resolve/main/granite-4.1-8b-Q4_K_M.gguf` |
| | Ornith-1.0-9B *(alt)* | Q4_K_M | 5.3 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/ornith-1.0-9b-Q4_K_M.gguf https://huggingface.co/deepreinforce-ai/Ornith-1.0-9B-GGUF/resolve/main/ornith-1.0-9b-Q4_K_M.gguf` |
| **Daily 📋** | **Gemma-4-12B-it** *(primary)* | Q4_K_M | 7.2 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/gemma-4-12b-it-Q4_K_M.gguf https://huggingface.co/bartowski/gemma-4-12B-it-GGUF/resolve/main/gemma-4-12b-it-Q4_K_M.gguf` |
| | Qwen2.5-14B-Instruct *(alt)* | Q4_K_M | 8.4 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/Qwen2.5-14B-Instruct-Q4_K_M.gguf https://huggingface.co/bartowski/Qwen2.5-14B-Instruct-GGUF/resolve/main/Qwen2.5-14B-Instruct-Q4_K_M.gguf` |
| | Gemma-4-12B-Uncensored *(uc)* | Q4_K_M | 6.9 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/gemma-4-12b-it-uncensored-Q4_K_M.gguf https://huggingface.co/zaakirio/gemma-4-12b-it-uncensored-GGUF/resolve/main/gemma-4-12b-it-uncensored-Q4_K_M.gguf` |
| | Gemma4-12B-QAT-Uncensored-Balanced *(uc)* | Q4_K_M | 6.9 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/Gemma4-12B-QAT-Uncensored-HauhauCS-Balanced-Q4_K_M.gguf https://huggingface.co/HauhauCS/Gemma4-12B-QAT-Uncensored-HauhauCS-Balanced/resolve/main/Gemma4-12B-QAT-Uncensored-HauhauCS-Balanced-Q4_K_M.gguf` |
| | Gemma-4-E4B-Uncensored-Aggressive *(uc)* | Q4_K_M | 5.0 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-Q4_K_M.gguf https://huggingface.co/HauhauCS/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive/resolve/main/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-Q4_K_M.gguf` |
| | Gemma-4-E4B-Heretic *(uc)* | Q4_K_M | 5.0 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/gemma-4-E4B-it-ultra-uncensored-heretic-Q4_K_M.gguf https://huggingface.co/llmfan46/gemma-4-E4B-it-ultra-uncensored-heretic-GGUF/resolve/main/gemma-4-E4B-it-ultra-uncensored-heretic-Q4_K_M.gguf` |
| **Think 🧠** | **DeepSeek-R1-Distill-Qwen-14B** *(primary)* | Q4_K_M | 8.4 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/deepseek-r1-distill-qwen-14b-Q4_K_M.gguf https://huggingface.co/bartowski/DeepSeek-R1-Distill-Qwen-14B-GGUF/resolve/main/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf` |
| | Qwen3-14B *(alt, thinking mode)* | Q4_K_M | 8.4 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/Qwen3-14B-Q4_K_M.gguf https://huggingface.co/Qwen/Qwen3-14B-GGUF/resolve/main/Qwen3-14B-Q4_K_M.gguf` |
| | Qwen3.5-9B-Uncensored-Aggressive *(uc)* | Q4_K_M | 5.3 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/Qwen3.5-9B-Uncensored-HauhauCS-Aggressive-Q4_K_M.gguf https://huggingface.co/HauhauCS/Qwen3.5-9B-Uncensored-HauhauCS-Aggressive/resolve/main/Qwen3.5-9B-Uncensored-HauhauCS-Aggressive-Q4_K_M.gguf` |
| | Qwen2.5-14B-Uncensored *(uc)* | Q4_K_M | 8.4 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/Qwen2.5-14B_Uncencored-Q4_K_M.gguf https://huggingface.co/bartowski/Qwen2.5-14B_Uncencored-GGUF/resolve/main/Qwen2.5-14B_Uncencored-Q4_K_M.gguf` |
| **Coder 🏋️** | **Qwen3.6-27B** *(primary)* | IQ4_XS | 15 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/qwen3.6-27b-IQ4_XS.gguf https://huggingface.co/unsloth/Qwen3.6-27B-GGUF/resolve/main/Qwen3.6-27B-IQ4_XS.gguf` |
| | Qwen3.6-27B *(alt)* | IQ4_NL | 15 GB | `curl -L -H "Authorization: Bearer \$HF_TOKEN" -o ~/models/Qwen3.6-27B-IQ4_NL.gguf https://huggingface.co/unsloth/Qwen3.6-27B-GGUF/resolve/main/Qwen3.6-27B-IQ4_NL.gguf` |

**14 models total, 110 GB on disk.** Unlabeled models are standard instruct;
*(uc)* = community uncensored variant. See `model-explorer` skill for download
and testing methodology. curl is zero-dependency; no `huggingface-cli` or pip
packages needed.

#### Starting the Router (Concrete Example)

```bash
source /opt/intel/oneapi/setvars.sh --force >/dev/null 2>&1 || true
export ONEAPI_DEVICE_SELECTOR=level_zero:0
llama-server \
    --port 8080 \
    --models-dir ~/models \
    --models-autoload \
    --sleep-idle-seconds 300 \
    --n-gpu-layers 99 \
    --host 0.0.0.0 \
    --ctx-size 98304
```

- `source setvars.sh` — sets `LD_LIBRARY_PATH` and oneAPI runtime env (required for SYCL/GPU)
- `--models-dir ~/models` — directory containing all GGUF files
- `--models-autoload` — models load on first API request
- `--sleep-idle-seconds 300` — unload idle models after 5 minutes
- `--n-gpu-layers 99` — offload all layers to GPU (llama.cpp b9828+ uses `--n-gpu-layers`, not `-ngl`; older versions used `-ngl`)

#### Testing Each Model

```bash
# Fast
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"granite-4.1-8b","messages":[{"role":"user","content":"Parse /var/log/pacman.log and summarize the last 5 package updates"}]}'

# Daily  
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"gemma-4-12b-it","messages":[{"role":"user","content":"Analyze this kernel log for hardware issues"}]}'

# Think
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"deepseek-r1-14b","messages":[{"role":"user","content":"Why would systemd-random-seed.service fail silently on a dual-LUKS setup?"}]}'

# Coder
curl http://localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3.6-27b","messages":[{"role":"user","content":"Refactor this 200-line shell script to handle errors properly"}]}'
```

#### Execution Steps (for This Host)

1. **Install the SYCL runtime:**
   ```bash
   sudo pacman -S intel-oneapi-toolkit
   ```
2. **Build llama.cpp with SYCL support** from AUR (see aur-build skill):
   ```bash
   # Clone and build llama.cpp-sycl-f16
   ```
3. **Download the 4 primary models** (~34 GB total):
   ```bash
   mkdir -p ~/models
   huggingface-cli install  # one-time tool install
   # Run the 4 download commands from the table above
   ```
4. **Start the router** (test manually first):
   ```bash
   source /opt/intel/oneapi/setvars.sh --force >/dev/null 2>&1 || true
   export ONEAPI_DEVICE_SELECTOR=level_zero:0
   llama-server --port 8080 --models-dir ~/models \
        --models-autoload --sleep-idle-seconds 300 --n-gpu-layers 99 --ctx-size 98304 --host 0.0.0.0
   ```
5. **Verify** each model responds via curl
6. **Configure opencode.jsonc** with multi-provider setup
7. **Create systemd service** for auto-start (optional)

---

## 4. Our Concrete Example: llama.cpp SYCL on Intel Arc

This section walks through a specific setup as a worked example. Replace hardware details with your own.

### 4.1 Hardware

| Component | Detail |
|-----------|--------|
| GPU | Intel Arc Pro B50 (Battlemage G21) — 16 GB GDDR6, 128 XMX engines |
| Kernel driver | `xe` |
| OS | Arch Linux |
| SYCL runtime | `intel-oneapi-toolkit` (Level Zero GPU backend) |
| llama-server | `llama.cpp-sycl` b9828-1 (AUR, built from `~/aur/llama.cpp-sycl/`) |

### 4.2 Packages installed

```bash
sudo pacman -S intel-oneapi-toolkit    # ~9 GB, full Level Zero + DPC++ compiler + MKL
```

Note: Arch's `intel-oneapi-dpcpp-cpp` is OpenCL-only — cannot target Intel Arc GPUs via Level Zero. The full toolkit is required.

### 4.3 Build and install

```bash
cd ~/aur/llama.cpp-sycl
makepkg -src                           # build
sudo pacman -U llama.cpp-sycl-*.pkg.tar.zst  # install
git clean -dfx                         # cleanup
```

Verified GPU: `llama-server --list-devices` shows `Intel(R) Arc(TM) Pro B50 Graphics (16304 MiB)`.

### 4.4 Download models

See section 3.6 for specific model recommendations. For this host, download the four primary models (~35 GB total):

```bash
mkdir -p ~/models
# Fast: Granite 4.1 8B (5.0 GB)
curl -L -o ~/models/granite-4.1-8b-Q4_K_M.gguf \
  'https://huggingface.co/ibm-granite/granite-4.1-8b-GGUF/resolve/main/granite-4.1-8b-Q4_K_M.gguf?download=1'
# Daily: Gemma 4 12B (7.2 GB)
curl -L -o ~/models/gemma-4-12b-it-Q4_K_M.gguf \
  'https://huggingface.co/bartowski/gemma-4-12B-it-GGUF/resolve/main/gemma-4-12B-it-Q4_K_M.gguf?download=1'
# Think: DeepSeek R1 Distill Qwen 14B (8.4 GB)
curl -L -o ~/models/deepseek-r1-distill-qwen-14b-Q4_K_M.gguf \
  'https://huggingface.co/bartowski/DeepSeek-R1-Distill-Qwen-14B-GGUF/resolve/main/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf?download=1'
# Coder: Qwen 3.6 27B IQ4_XS (15 GB, ~1 GB free for KV cache)
curl -L -o ~/models/qwen3.6-27b-IQ4_XS.gguf \
  'https://huggingface.co/unsloth/Qwen3.6-27B-GGUF/resolve/main/Qwen3.6-27B-IQ4_XS.gguf?download=1'
```

### 4.5 Start the server

Router mode (multiple models on one port, auto-load/auto-unload):

```bash
source /opt/intel/oneapi/setvars.sh --force >/dev/null 2>&1 || true
export ONEAPI_DEVICE_SELECTOR=level_zero:0
llama-server \
    --port 8080 \
    --models-dir ~/models \
    --models-autoload \
    --sleep-idle-seconds 300 \
    --n-gpu-layers 99 \
    --host 0.0.0.0 \
    --ctx-size 98304
```

- `source setvars.sh` — sets oneAPI `LD_LIBRARY_PATH` + runtime env (required for SYCL/GPU detection)
- `ONEAPI_DEVICE_SELECTOR=level_zero:0` — tells oneAPI to use the first Intel GPU (Level Zero backend)
- `--n-gpu-layers 99` — offloads as many layers as possible to the GPU; llama.cpp will fit what it can in VRAM
- `--ctx-size 98304` — 96K context window; adjust based on available VRAM

The server logs startup progress. The first launch is slow (JIT compilation); subsequent starts are faster.

### 4.6 Configure opencode

Add to `~/.config/opencode/opencode.jsonc`:

```jsonc
{
    "model": {
        "provider": "openai-compatible",
        "name": "qwen3.5-14b",
        "base_url": "http://localhost:8080/v1"
    }
}
```

Verify it works:
```bash
curl http://localhost:8080/v1/models
```

### 4.7 Systemd service (optional)

To auto-start the model server:

`~/.config/systemd/user/llama-server.service`:
```ini
[Unit]
Description=llama.cpp inference server (model pool, Intel Arc GPU)
After=network.target

[Service]
Type=simple
# setvars.sh sets the oneAPI LD_LIBRARY_PATH + runtime env (required for SYCL/GPU)
ExecStart=/usr/bin/bash -c 'source /opt/intel/oneapi/setvars.sh --force >/dev/null 2>&1 || true; export ONEAPI_DEVICE_SELECTOR=level_zero:0; exec /usr/bin/llama-server --models-dir %h/models --port 8080 --n-gpu-layers 99 --ctx-size 98304 --sleep-idle-seconds 300'
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
chmod +x ~/.local/bin/opencode-health-check
systemctl --user daemon-reload
systemctl --user enable --now llama-server.service
```

---

## 5. Skills Reference

Skills are reusable instruction sets the domovoy loads for specific tasks. They live in `~/.agents/skills/` and are synced across machines.

| Skill | When to load |
|-------|-------------|
| `maintenance-report` | Start of every session — binlog format and rules |
| `system-documentation` | When updating SYSTEM_INFO or SYSTEM_SETUP — explains the three-document system |
| `system-info` | When onboarding a new machine — guides populating SYSTEM_INFO.md |
| `aur-build` | When installing or updating AUR packages — manual workflow, no helpers |

**Example — our loaded skills:**
```
maintenance-report    → every session
system-documentation  → document updates
system-info           → machine bootstrap
aur-build             → AUR package tasks
```

---

## 6. Document Boundaries

Multiple documents track the system. Know what goes where.

| Document | Answers | Updated when |
|----------|---------|-------------|
| **SYSTEM_SETUP.md** | "How to rebuild this machine from bare metal" | Permanent config changes |
| **DOMOVOY_SETUP.md** (this document) | "How to configure the AI agent across machines" | Agent-specific changes |
| **SYSTEM_INFO.md** | "What is this machine right now?" | Hardware, kernel, package changes |
| **maintenance/reports/** | "What did we actually do?" | Every state-changing command |
| **MIGRATION.md** | "How to deploy this to a new machine" | Bootstrap process updates |

**SYSTEM_SETUP covers:** Partitioning, encryption, bootloader, packages, services, timers, kernel parameters, btrfs layout, snapper config. — everything needed to reproduce the OS.

**DOMOVOY_SETUP covers:** Assistant user, opencode service, model provider decisions, runner setup, skills. — the AI agent layer on top of the OS.

**Do NOT put in DOMOVOY_SETUP:** Machine-specific UUIDs, IP addresses, internal paths containing hostnames, Syncthing device IDs, API keys. This document is meant to be shared.
