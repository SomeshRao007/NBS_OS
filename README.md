# NBS OS

**A Debian-based Linux distribution with a fine-tuned LLM built into the operating system.**

Ask a sysadmin question in plain English, get the correct command — running entirely on-device, no API key, no network, ~4 GB VRAM.

[![model](https://img.shields.io/badge/%F0%9F%A4%97%20model-qwen__os__ai-yellow)](https://huggingface.co/SomeshRao007/qwen_os_ai)
[![accuracy](https://img.shields.io/badge/harness%20accuracy-98.8%25-brightgreen)]()
[![base](https://img.shields.io/badge/base-Qwen3.5--4B-blue)]()

```
neurosh> ? find files over 100MB modified in the last week

  find / -type f -size +100M -mtime -7
```
  Searches from root for files larger than 100MB modified within 7 days.

<!-- TODO: replace with a real demo GIF — boot the ISO in a VM, open the Plasma
     widget, ask a question with wifi visibly off. Put it at docs/demo.gif -->
<!-- ![demo](docs/demo.gif) -->

---

## Why

General-purpose assistants know Linux, but you have to leave the terminal, paste
context, and trust a cloud endpoint with your system state. NBS OS puts a
purpose-trained 4B model *inside* the OS: it knows your distro, your working
directory and your session history, it never leaves the machine, and every
command it produces is validated and sandboxed before it can touch anything.

The interesting problem was never "call an LLM." It was making a 4B model on
consumer hardware reliable enough to trust with shell commands. That took a
scaffolding harness around the model — routing, retrieval, validation,
sandboxing — which moved end-to-end accuracy from **88% to 98.8%**.

---

## Results

### The harness is worth more than the model

Each layer added to the raw fine-tuned model, measured on the same 164-question
evaluation set:

| Stage | Accuracy |
|---|---|
| Raw fine-tuned model | 88.0% |
| + system prompts | ~91% |
| + RAG injection + validator | 96.0% |
| + full harness (all 10 components) | **98.8%** (162/164) |

16 of 18 domains score 100%. The two misses are `debugging` (87.5%) and
`systemd` (80%).

### Synthetic data ablation

Two identical QLoRA runs, differing only in whether the 23k synthetic dataset
was included:

| Metric | No synthetic | +23k synthetic | Δ |
|---|---|---|---|
| Eval loss | 0.7736 | **0.5978** | −23% |
| Training loss | 0.7346 | 0.6611 | −10% |
| Eval token accuracy | 83.3% | **86.8%** | +3.5 pp |
| Dataset | 18.8M tokens | 23.8M tokens | +27% |
| Steps / epochs | 585 / 3 | 756 / 3 | — |
| Runtime | 14.4 h | 16.3 h | +1.9 h |

### Runtime performance

| Metric | Value |
|---|---|
| Throughput | 51–70 tok/s |
| Avg response latency | 672 ms |
| Model VRAM | 3,292 MB |
| Peak VRAM | 4,082 MB |
| Quantization speedup | 25.5 → 70.3 tok/s (2.8× over the 4-bit adapter) |

### Stability

- **Memory:** 100 sequential queries — VRAM flat (−47 MB), RSS +112 MB, no leak.
- **Context rot:** 50-query session — answer quality 3.00 → 2.72 / 3.0, latency
  +75 ms. Injected context plateaus at 84 tokens (4.1% of the 2048 window); no
  runaway growth.
- **Routing:** keyword classification alone resolves 90.9% of queries at zero
  inference cost; with model fallback, 93.2%.

Full result sets live next to the code they measure — see
[`os_agent/Agent_benchmark_testing/`](os_agent/Agent_benchmark_testing/).

---

## Architecture

```
┌─ ISO (Debian live-build) ────────────────────────────────────────────┐
│                                                                       │
│   KDE Plasma widget          neurosh              Settings app        │
│   (org.aios.assistant)       (terminal │ chatbot │ ai)   (PySide6)    │
│           │                      │                      │             │
│           └──────────────── D-Bus (org.aios.Daemon) ─────┘            │
│                                  │                                    │
│   ┌──────────────────────────────┴─────────────────────────────────┐  │
│   │  ai-daemon  (systemd user service, MemoryMax=8G)               │  │
│   │                                                                 │  │
│   │  MasterAgent ── 2-stage routing ──┐                             │  │
│   │    keywords (free) → model (ties) │                             │  │
│   │                                    ▼                            │  │
│   │        files · network · process · packages · kernel            │  │
│   │                                    │                            │  │
│   │  Memory ─── SharedState (action log, 50 entries)                │  │
│   │         ├── AgentMemory (FAISS, 384-d MiniLM, 500 vec/domain)   │  │
│   │         └── SessionContext (20-turn FIFO, ~84 tok)              │  │
│   │                                    │                            │  │
│   │  RAG ────── keyword → help_db.json (208 command schemas)        │  │
│   │  Validator ─ flags, deprecated syntax, arg types, distro        │  │
│   │  Executor ── bubblewrap sandbox, 3-tier risk, 30s timeout       │  │
│   │                                    │                            │  │
│   │  BackendManager ─┬─ llama.cpp (local GGUF)   ← default          │  │
│   │                  └─ OpenRouter (cloud, opt-in, KWallet key)     │  │
│   └─────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│   Model: qwen3.5-4b-os-q4km.gguf (2.6 GB, bundled)                     │
└────────────────────────────────────────────────────────────────────────┘
```

### Command safety model

Nothing the model generates reaches a shell unchecked. Commands are parsed out
of the response, validated against real `--help` schemas, classified by risk,
then run inside a bubblewrap sandbox (`--die-with-parent`, 30 s timeout).

| Tier | Behaviour | Examples |
|---|---|---|
| **Safe** (61 commands) | auto-execute | `ls`, `cat`, `df`, `ps`, `git status` |
| **Moderate** | y/n confirmation | `mkdir`, `cp`, `apt install`, `git commit` |
| **Dangerous** (18 regex patterns) | y/n + red warning | `rm -rf`, `mkfs`, `dd`, `kill -9`, `shutdown` |
| **Out-of-domain** | y/n + domain warning | anything outside the agent's whitelist |

Out-of-domain commands are flagged, not blocked — the agent may have suggested
something useful outside its normal scope, and the user decides. `git` is
classified per-subcommand (14 read-only subcommands are safe, writes are
moderate).

The validator catches what a language model gets wrong most often: invalid or
unknown flags, deprecated syntax (`find -perm +644` → `/644`), argument type
mismatches (an email where a filepath belongs), duplicate flags, and
wrong-distro package managers (`dnf` on Debian — warned, never blocked).

---

## Repository layout

```
finetuning/          Data generation, QLoRA training, GGUF export, evals
  data/                collect · generate_synthetic · filter · reformat
  configs/             qlora_config.yaml — single source of truth
  q4_k_m-deploy/       merge → GGUF export, Ollama Modelfile
  training_data/       W&B run histories (ablation)
  Testing_models/      adapter vs GGUF comparison

os_agent/            The daemon
  agents/              MasterAgent + 5 domain specialists
  inference/           backends, engine, RAG, validator, prompts, registry
  memory/              3-tier memory + embedded MiniLM
  tools/               registry, sandboxed executor, parser, help-db builder
  shell/               neurosh — modes, completer, history, renderer
  ipc/                 D-Bus service, unix socket, client
  Agent_benchmark_testing/   5 benchmark suites

os_build/            ISO construction
  build.sh             live-build orchestration
  Dockerfile.builder   reproducible build container
  config/              package lists, chroot includes, systemd units
  plasmoids/           KDE Plasma widget (QML)
  settings/            PySide6 settings app
```

---

## Quickstart

### 1. Run the agent

```bash
git clone https://github.com/SomeshRao007/NBS_OS.git && cd NBS_OS
python3 -m venv .venv && source .venv/bin/activate

# CPU
pip install -r requirements.txt

# NVIDIA GPU (recommended — the model is sized for ~4 GB VRAM)
CMAKE_ARGS="-DGGML_CUDA=on" pip install --force-reinstall --no-cache-dir llama-cpp-python
pip install -r requirements.txt
```

The GGUF model is not in git (2.6 GB). Pull it from Hugging Face:

```bash
pip install huggingface_hub
hf download SomeshRao007/qwen_os_ai qwen3.5-4b-os-q4km.gguf \
  --local-dir finetuning/q4_k_m-deploy/
```

Or run it standalone under Ollama — the repo ships a `Modelfile`:

```bash
ollama create os-ai -f Modelfile && ollama run os-ai "list all running services"
```

Then start the agent:

```bash
python -m os_agent            # neurosh interactive shell
python -m os_agent --daemon   # D-Bus daemon (systemd mode)
```

In neurosh: `?` prefix sends a line to the agent, `!` forces a raw shell
command, `/` runs a meta-command. Three modes — `terminal`, `chatbot`, and `ai`
(co-pilot: context-aware agent plus execution).

Install `bubblewrap` to enable sandboxed execution — without it the daemon logs
a warning and runs commands directly.

### 2. Reproduce the model

Training was run on a single rented GPU (Vast.ai), ~16 h for 3 epochs.

```bash
cd finetuning
pip install -r requirements-train.txt anthropic
python validate_setup.py                    # check GPU, deps, dataset

export ANTHROPIC_API_KEY=...                # synthetic generation, ~$8
python data/generate_synthetic.py           # 23k examples, 19 Linux domains
python data/filter_data.py \
  --input data/raw/combined_raw.jsonl \
  --split                                   # dedup, credential + syntax + safety filters

python train_trl.py --config configs/qlora_config.yaml
bash q4_k_m-deploy/export_q4km.sh output/qlora-run   # merge → GGUF Q4_K_M
```

`train_trl.py` accepts CLI overrides for any config value — `--epochs`, `--lr`,
`--batch-size`, `--max-steps` (dry runs), `--resume`, `--wandb`.

All hyperparameters live in [`configs/qlora_config.yaml`](finetuning/configs/qlora_config.yaml)
— rank 64, α 128, NF4 double-quant, paged AdamW 8-bit, bf16, gradient
checkpointing, cosine schedule. CLI flags override any value in the file.

### 3. Build the ISO

Requires Docker. The build needs ~20 GB free disk and produces a ~5 GB ISO.

```bash
cd os_build
./docker-build.sh                          # default bundled model
./docker-build.sh --model /path/to/x.gguf  # swap in a different model
# → os_build/output/*.iso
```

Nothing is installed on the host — all live-build tooling runs inside a Debian
13 container. Test the result without writing a USB stick:

```bash
qemu-system-x86_64 -enable-kvm -m 4G -smp 2 \
  -cdrom os_build/output/*.iso -boot d -vga virtio
```

The daemon starts automatically as a systemd user service once the graphical
session is up.

---

## Configuration

`os_agent/config/daemon.yaml` — model path, generation parameters, shell
prefixes, memory limits, sandbox, notifications.
`os_agent/config/models.yaml` — model registry. Local GGUF entries and optional
OpenRouter cloud inference; API keys are stored in KWallet, never in the file.

Generation defaults were chosen by a 4-config parameter sweep over 44 questions:
`temperature 0.3`, `top_p 0.98`, `top_k 15`, `repeat_penalty 1.1`. Lower
temperature eliminated hallucinated flags; `repeat_penalty` fixed duplicate-flag
output (`useradd -m -m`). A 4096-token context showed no accuracy change over
1024 but leaves headroom for session-memory injection.

---

## Status

Working end to end: data pipeline, training, quantized export, daemon, routing,
memory, RAG, validation, sandboxed execution, neurosh, Plasma widget, settings
app, and a bootable ISO.

Known gaps:

- `debugging` and `systemd` are the weakest domains (87.5% / 80%).
- D-Bus `Query` streams via a single reply; per-token `ResponseChunk` signals
  are scaffolded but not yet wired to the QML chat UI.
- Network isolation in the sandbox (`--unshare-net`) is disabled for
  unprivileged users; it needs `CAP_NET_ADMIN`.
- No published ISO release — build locally for now.

---

## Model

The fine-tuned model is released separately at
**[SomeshRao007/qwen_os_ai](https://huggingface.co/SomeshRao007/qwen_os_ai)** —
Qwen3.5-4B QLoRA fine-tune, merged and quantized to GGUF Q4_K_M (2.6 GB),
Apache-2.0. The repo includes Ollama `Modelfile`s (standard and
thinking-enabled), the export script, and the GGUF eval harness.

## License

<!-- TODO: add a LICENSE file to this repo. -->
Code: MIT — see [LICENSE](LICENSE).
Model weights: Apache-2.0, per the
[model card](https://huggingface.co/SomeshRao007/qwen_os_ai).
