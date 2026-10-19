# PROJECT ARCHITECTURE BLUEPRINT: ALFREDSBRAIN (formerly STATION75)
## Core Objective: Self-Hosted Neural Sandbox & Offline Knowledge Vault

## About the operator
I'm new to Linux, servers, and AI. Before running any command, explain in plain English what it does and why. Ask before anything that uses sudo, changes the firewall, or touches drivers. Work one small step at a time.

---

## BUILD ORDER (PHASES)
*   **Phase 0 (done by operator):** Fresh Ubuntu 24.04 LTS Desktop install on the Samsung SSD, system updated, `git` and `openssh-server` installed, DHCP reservation set on the router.
*   **Phase 1: Host Setup.** Propose separate, re-runnable scripts, one per concern, in this order:
    1. Install and pin the NVIDIA R580 proprietary driver (Section 1), reboot, confirm both GPUs with `nvidia-smi`.
    2. Firewall: allow SSH (port 22) from the LAN subnet **before** running `ufw enable`, then add the remaining rules from Section 2.
    3. Install Ollama, apply the systemd override (Section 2), pull the three models, create `llama3.2-cpu` (Section 4), and verify placement with `ollama ps`.
    4. Create the project folder layout and offline caching paths (Section 3).
*   **Phase 2: Survival Oracle.** Build the ingestion script, vector storage, and Docker Compose frontend (Section 4, Profile B).
*   **Phase 3: Textbook Sandbox.** Run the offline pre-staging script, then work through the book chapter by chapter, starting with Chapter 2 (tokenizer and text data). Self-attention comes in Chapter 3.

---

### 1. TARGET HARDWARE PROFILE & RUNTIME TUNING
*   **CPU:** Intel Core i7-4790K (4 cores / 8 threads, 4.0GHz)
    *   *Optimization directive:* Limit LLM workloads to **4 threads** (`torch.set_num_threads(4)` and matching Ollama parameters). Do not disable hyperthreading in the system BIOS, as secondary workloads (like Jellyfin transcoding) need thread availability.
*   **System RAM:** 24GB DDR3-1600 (Dual-channel bandwidth yields approx. 8-12 tokens/sec on 3B Q4 models).
*   **Storage:** Samsung 850 EVO 250GB SSD + SATA expansion drives.
*   **GPU 0 (Quadro K2200 4GB VRAM):** Assigned exclusively to Jellyfin NVENC transcoding.
*   **GPU 1 (Quadro K2200 4GB VRAM):** Assigned to hardware-accelerated Ollama code generation.
    *   *Microarchitecture profile:* Maxwell (Compute Capability 5.0). Recent PyTorch CUDA builds have dropped Maxwell support, so all textbook code runs in CPU mode. Ollama still supports compute 5.0 cards.
    *   *Driver restriction:* The NVIDIA R580 branch is the final driver series supporting Maxwell on Linux. Pin it on Ubuntu and lock updates via `sudo apt-mark hold nvidia-driver-580`.
    *   *CRITICAL INSTALLATION WARNING:* NVIDIA's open-source kernel modules do not support Maxwell. Install the proprietary package `nvidia-driver-580` and **strictly avoid** `nvidia-driver-580-open`.
    *   *VRAM headroom constraint:* GPU 1 also drives the Ubuntu desktop, which uses part of its 4GB VRAM. Run compact models only.

---

### 2. LOCAL ENGINE PIPELINE BOUNDARIES & NETWORKING
Run the core AI stack directly on the host; use containers only for the Oracle app.
*   **Native Host Execution:** Ollama runs natively as a systemd service for direct access to the Maxwell GPU and CPU, without `nvidia-container-toolkit`.
*   **Isolate GPU 1 by UUID:** Get the second Quadro K2200's UUID via `nvidia-smi -L`. In the systemd override, set `CUDA_VISIBLE_DEVICES=GPU-XXXXXXXX-XXXX-...` so the assignment survives reboots.
*   **Environment Limits (`/etc/systemd/system/ollama.service.d/override.conf`):**
    *   `OLLAMA_NUM_PARALLEL=1` (one request at a time).
    *   `OLLAMA_MAX_LOADED_MODELS=3` (the coding assistant, the RAG answer engine, and the embedding model stay loaded together; 24GB RAM handles all three).
*   **Docker Container Rules:** Use Docker Compose only for the Oracle frontend and its document database.
*   **Docker-to-Ollama Gateway Bridge:** The Oracle container cannot reach Ollama at `localhost:11434`. Add `extra_hosts: ["host.docker.internal:host-gateway"]` to the Oracle service in `docker-compose.yml` and point it at `http://host.docker.internal:11434`.
*   **UFW Firewall Rules:**
    1. Docker bypasses UFW when publishing ports. Bind every published port in `docker-compose.yml` to the static LAN IP (e.g., `ports: ["192.168.1.100:8080:8080"]`). ALFREDSBRAIN has a DHCP reservation; if its IP changes, Docker will refuse to start the container.
    2. UFW blocks container-to-host traffic by default. Allow the Docker bridge subnet (typically `172.16.0.0/12`) to reach port `11434`, or the frontend will time out.
    3. Set `OLLAMA_HOST=0.0.0.0` and restrict inbound access with UFW rules limited to the local network subnet.

---

### 3. CORE GOAL 1: TEXTBOOK ENGINE CODEBUILD (CLAUDE CODE DIRECTIVES)
**Source Text:** *Build a Large Language Model (From Scratch)* by Sebastian Raschka.

#### Strict Build Directives for Claude Code:
1.  **Low-Level Execution Rules:** Build every mechanic, including multi-head causal self-attention and layer normalization, line by line using vanilla PyTorch tensors. **Do not use HuggingFace Transformers or other abstraction layers.**
2.  **Tokenizer Strategy:** Hand-code a simple word-splitting tokenizer as taught in the early chapters. Use the `tiktoken` library for Byte-Pair Encoding (BPE) in later chapters. Do not rewrite BPE from scratch.
3.  **Hardware Binding & Model Size:** All textbook training and fine-tuning scripts use `device = torch.device('cpu')`. Models trained from scratch are capped at **10M to 50M parameters**. Loading the pretrained GPT-2 **124M** weights for generation and fine-tuning in Chapters 5–7 is explicitly allowed.
4.  **True Offline Pre-Staging:** The textbook downloads `tiktoken` encoding files, GPT-2 weights, the spam dataset (Chapter 6), and the instruction dataset (Chapter 7) at runtime.
    *   *Ollama Model Storage:* Ollama keeps models in its own store (`/usr/share/ollama/.ollama/models`). Do not put Ollama files in the repo. While online, run `ollama pull` for `qwen2.5-coder:1.5b`, `llama3.2:3b`, and `nomic-embed-text`. Back up that directory separately.
    *   *Evaluation Note (Chapter 7):* The textbook scores the fine-tuned model with a larger model through Ollama. Substitute the CPU-pinned `llama3.2-cpu` model and accept rougher scoring. Expect this step to be slow.
    *   *Caching layout:* Set `TIKTOKEN_CACHE_DIR` to a folder inside the repo. Run the download steps once while online, saving the BPE files, GPT-2 124M weights, and datasets into local `models/` and `data/` directories. Back these up so everything runs 100% offline afterward.

---

### 4. CORE GOAL 2: DUAL-PURPOSE OFFLINE WORKSPACE APPLICATIONS

#### Profile A: Private Software Development Assistant
*   **Execution Target:** Load `qwen2.5-coder:1.5b` (Q4, ~1GB) fully into GPU 1's VRAM.
*   **OpenAI-Compatible API:** Expose Ollama's `/v1` route at `http://<ALFREDSBRAIN_LAN_IP>:11434/v1` so other machines on the LAN can use ALFREDSBRAIN for code completion, script reviews, and SQL help without sending files to cloud providers.

#### Profile B: Home Survival & Grid-Down Knowledge Vault
*   **The Oracle Loop:** An offline Retrieval-Augmented Generation (RAG) tool that uses the custom `llama3.2-cpu` model for answers.
    *   *Text Extraction:* Use `PyMuPDF` or `pdftotext` in the ingestion script to pull text from local survival PDFs.
    *   *Embeddings:* Use `nomic-embed-text` through native Ollama.
    *   *Storage:* Store and query vectors with NumPy arrays or `sqlite-vec`.
    *   *Forced Model Placement Modelfile:* Create a distinct **`llama3.2-cpu`** model so the answer engine stays off GPU 1. Build it with `ollama create llama3.2-cpu -f Modelfile`, using:
        ```
        FROM llama3.2:3b
        PARAMETER num_gpu 0
        PARAMETER num_thread 4
        PARAMETER num_ctx 2048
        ```
    *   *Verification & API Safeguards:*
        1. After building the model, run it once and check `ollama ps`. The `PROCESSOR` column must read `100% CPU`. Repeat after every Ollama update.
        2. `num_gpu` is no longer listed in Ollama's Modelfile documentation, although it still works. As a backup, the Oracle backend must also send `"options": {"num_gpu": 0, "num_thread": 4}` with every API request.
*   **Prompt Size Limits:** CPU prompt processing is the main source of delay. Cap each request at **3 retrieved chunks** and **1,000 tokens** of retrieved context.
*   **Safety Rule #1 (Source Verification):** The Oracle frontend must show the exact, unsummarized source passage next to every answer. Small 3B models confidently invent first-aid or wiring steps, so the operator must be able to check the original text instantly.
*   **Safety Rule #2 (Analog Redundancy):** ALFREDSBRAIN draws significant power. In a grid-down or low-power situation it is too resource-intensive to run. Keep the original source documents readable on low-power battery devices or in print so survival information never depends on the server.
