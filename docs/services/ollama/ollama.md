# Ollama - Self Hosted AI 

[Github :simple-github: ](https://github.com/ollama/ollama)

---

## Ollama Installation

---

Official script:

This will auto detect hardware and install necessary dependancies 

`curl -fsSL https://ollama.com/install.sh | sh`

To run a model, use:

`ollama run gemma3` or `ollama run qwen2.5-coder:14b`

##  Models 

---

Current Best models would be: 


Here is the table formatted in Markdown.

| Category | Model | Size | Speed | Use Case |
| --- | --- | --- | --- | --- |
| **Coding** | **Qwen 2.5 Coder 32B** | 19GB | Medium | Complex scripts, Docker, Python, Arch Linux help. |
| **Logic** | **DeepSeek R1 32B** | 19GB | Medium | Debugging weird errors, math, logic puzzles. |
| **Writing** | **Gemma 2 27B** | 16GB | Fast/Med | Emails, documentation, stories, cover letters. |
| **Speed** | **Llama 3.1 8B** | 5GB | Instant | Quick facts, summarization, simple chat. |

### Quick Pull List

You can copy and paste this block to download them all at once:

```bash
ollama pull qwen2.5-coder:32b
ollama pull deepseek-r1:32b
ollama pull gemma2:27b
ollama pull llama3.1

```

## Open-WebUI

---

For a nice GUI, you can use Open-WebUI.

This compose assumes you have ollama running on host machines. For other configs, check the docs

https://github.com/open-webui/open-webui

```yaml
### --- OpenWebUI - Interface for Ollama --- ###
# https://github.com/open-webui/open-webui
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: always
    network_mode: host
    #ports:
    # - "3000:8080"
    volumes:
      - ./open-webui-data:/app/backend/data
    extra_hosts:
      - host.docker.internal:host-gateway
```

Access this at http://localhost:8080.

!!!note
    To correctly show models in the WebUI, I had to manually set the connections (admin page > settings) for Ollama API as `http://127.0.0.1:11434`

## ROCm 

---

ROCm (Radeon Open Compute) is AMD's open-source software stack for GPU computing.

This is needed to use the GPU as compute instead of just for graphical reasons. 

**Installation** is here: https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/quick-start.html

**After installation**, to verift it is working run:

`watch -n 1 rocm-smi`  

This will monitor the GPU so you can watch it work. 

While monitoring, run: 

`ollama run qwen2.5-coder:14b "some fancy shit here in rust"`

You should see the GPU spike. 

**OR**

Check the boot logs. Ollama is pretty clear on what is being used.

`sudo journalctl -u ollama --no-pager | grep "rocm"`


## Optimization 

---

### xpand Context Window

By default, Ollama uses a **4,096 token** context window. This is too small for large code files or pasting logs.
6800 XT (16GB VRAM) has enough room to run the 14B model with a much larger "memory" of your conversation.

* **Goal:** Increase context to **16k** or **32k**.
* **Why:** Allows the model to see your entire config file or script at once.

**How to set it permanently:**

1. Edit the Ollama service:
```bash
sudo systemctl edit ollama.service

```


2. Add these environment variables to the `[Service]` section:
```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"
# Enable Flash Attention (Crucial for RDNA2 performance at high context)
Environment="OLLAMA_FLASH_ATTENTION=1"
# Set default context to 32k (fits in your 16GB VRAM with 14B model)
Environment="OLLAMA_NUM_CTX=32768"

```


3. Save, exit, and restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama

```



### Model Selection

You have 64GB RAM, which puts you in a unique position. You can run models larger than your GPU VRAM.

* **The "Daily Driver" (Fast & Capable):** `qwen2.5-coder:14b`
* **VRAM Usage:** ~9GB (Model) + ~2-4GB (Context). Fits 100% on GPU.
* **Speed:** Instant.
* **Use case:** General questions, writing small scripts, Linux commands.
* *Optimization:* Create a custom model file to lock in parameters (see below).


* **The "Heavy Lifter" (Maximum Accuracy):** `qwen2.5-coder:32b`
* **VRAM Usage:** ~20GB.
* **Your Setup:** 16GB on GPU + ~4GB spillover to System RAM.
* **Speed:** Slower (expect 4-8 tokens/sec), but **significantly** smarter at complex logic.
* **Use case:** Debugging a complex race condition, architecting a new system, or when 14B gets stuck.



**Recommendation:** Keep both. Use 14B for speed, switch to 32B when you need a "senior engineer" to look at your code.

### Model File

Don't just use the raw model. Create a custom "Modelfile" to bake in your preferences (Linux, Docker, minimal fluff).

1. In Open WebUI, go to **Workspace** (left sidebar) -> **Models** -> **Create a Model**.
2. **Name:** `Goose-Coder-14B`
3. **Base Model:** `qwen2.5-coder:14b`
4. **System Prompt:**
> You are an expert Linux flight simulation technician and DevOps engineer. You prefer "Gruvbox" aesthetics where applicable. You specialize in Docker, Self-Hosting, and System Administration.
> Guidelines:


> 1. When providing code, use strictly official documentation sources.
> 2. Do not use conversational filler ("Here is the code you asked for"). Just provide the solution.
> 3. Prefer `docker compose` over `docker run`.
> 4. Assume a modern Linux environment (Arch/Debian/Ubuntu).
> 
> 


5. **Advanced Params:**
* **Temperature:** `0.2` (Lower is better for code; prevents hallucinations).
* **Context Length:** `32768` (Ensure this matches your hardware capability).


6. **Save & Use.**


## Specific model locations

---

```bash
    sudo systemctl stop ollama
```

```bash
# Create the folder
sudo mkdir -p /mnt/fast_ssd/ollama_models

# (Optional) Move existing models if you already downloaded some
sudo mv /usr/share/ollama/.ollama/models/* /mnt/fast_ssd/ollama_models/

```

```bash
sudo chown -R ollama:ollama /mnt/fast_ssd/ollama_models
sudo chmod -R 775 /mnt/fast_ssd/ollama_models
```

Update systemd service:

```bash
sudo systemctl edit ollama.service
```

Add this to .service 

```bash
[Service]
Environment="OLLAMA_MODELS=/mnt/fast_ssd/ollama_models"
```

restart

```bash
sudo systemctl daemon-reload
sudo systemctl start ollama
```

Verify

```bash
ollama list

# pull new model
ollama pull tinyllama
```

Check folder 

```bash
ls -lh /mnt/fast_ssd/ollama_models/blobs
```