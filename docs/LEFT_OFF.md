# Where We Left Off
*Last updated: 2026-10-19*

## Status
Repo created and initial structure pushed. Book uploaded. Ready to begin Phase 1.

## Phase 0 — Done by operator
- [ ] Fresh Ubuntu 24.04 LTS Desktop install on Samsung 850 EVO 250GB SSD
- [ ] System updated
- [ ] `git` and `openssh-server` installed
- [ ] DHCP reservation set on router for ALFREDSBRAIN

**Confirm Phase 0 is complete before starting Phase 1.**

## Next Step — Phase 1: Host Setup
Work through these in order, one script at a time:

1. **NVIDIA driver** — Install and pin `nvidia-driver-580` (proprietary, NOT the `-open` variant). Reboot. Confirm both K2200s show up with `nvidia-smi`.
2. **Firewall** — Allow SSH (port 22) from LAN subnet first, then `ufw enable`, then add remaining rules.
3. **Ollama** — Install, apply systemd override, pull models (`qwen2.5-coder:1.5b`, `llama3.2:3b`, `nomic-embed-text`), create `llama3.2-cpu` Modelfile.
4. **Folder layout** — Create project and offline cache directories.

## Repo
- `BLUEPRINT.md` — full architecture reference
- `docs/Build_a_Large_Language_Model_From_Scratch_-_Sebastian_Raschka.pdf` — textbook for Phase 3
- `phase1-host-setup/` — scripts go here
- `phase2-oracle/` — survival RAG oracle
- `phase3-textbook/` — LLM from scratch (Raschka)
