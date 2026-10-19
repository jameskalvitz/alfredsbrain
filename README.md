# alfredsbrain

**Self-Hosted Neural Sandbox & Offline Knowledge Vault**

A fully offline AI workstation built on salvaged enterprise hardware. Runs local LLMs, a survival knowledge RAG oracle, a media server, and a from-scratch LLM textbook build — all on your own hardware, no cloud required.

---

## Hardware

| Component | Spec |
|---|---|
| CPU | Intel Core i7-4790K — 4c/8t, 4.0GHz (Turbo 4.4GHz) |
| RAM | 24GB DDR3-1600 |
| Storage | Samsung 850 EVO 250GB SSD |
| GPU 0 | NVIDIA Quadro K2200 4GB GDDR5 → Jellyfin NVENC |
| GPU 1 | NVIDIA Quadro K2200 4GB GDDR5 → Ollama LLM inference |
| PSU | Corsair CX750M 750W 80 PLUS Bronze |
| Motherboard | ASUS Z87-PRO |
| OS | Ubuntu 24.04 LTS |

---

## Project Structure

```
alfredsbrain/
├── BLUEPRINT.md           # Full architecture blueprint
├── phase1-host-setup/     # Driver, firewall, Ollama install scripts
├── phase2-oracle/         # Survival RAG oracle (Docker Compose + ingestion)
├── phase3-textbook/       # LLM from scratch (Raschka book)
│   ├── models/            # GPT-2 weights (offline cache)
│   ├── data/              # Datasets (spam, instruction)
│   └── cache/             # tiktoken BPE cache
├── scripts/               # Utility scripts
└── docs/                  # Notes, logs, decisions
```

---

## Phases

- **Phase 0** — Ubuntu install, SSH, DHCP reservation ✅ *(done by operator)*
- **Phase 1** — NVIDIA driver, UFW firewall, Ollama setup
- **Phase 2** — Survival Oracle (offline RAG)
- **Phase 3** — LLM textbook build (Raschka, Chapter by Chapter)

See [BLUEPRINT.md](BLUEPRINT.md) for full architecture details.
