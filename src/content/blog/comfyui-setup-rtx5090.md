---
title: "Running ComfyUI on RTX 5090 in Abu Dhabi — Local AI Image Generation Setup"
description: "How I set up ComfyUI with Z-Image-Turbo on an RTX 5090 for fast local image generation. No cloud, no API costs — full GPU power at home."
pubDate: 2026-03-15
---

I've been running **ComfyUI** locally on my RTX 5090 here in Abu Dhabi, and it's been a game changer. No API costs, no data leaving the machine, and generation times that make cloud services feel slow.

Here's exactly how I set it up.

## My Setup

- **GPU:** NVIDIA RTX 5090 (32GB VRAM)
- **RAM:** 64GB
- **OS:** Ubuntu 24
- **CUDA:** 12.8
- **Driver:** 570.211.01

The 5090's 32GB VRAM is overkill for most workflows — but it means I can run large models without compromise.

## What is ComfyUI?

ComfyUI is a node-based UI for Stable Diffusion. Instead of a simple prompt box, you build a **workflow graph** — nodes connected together representing each step of the generation pipeline. It's more powerful and more flexible than any other SD interface.

It also exposes a REST API, which means you can trigger generation programmatically — which is exactly what I use for automation.

## Installing ComfyUI

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI
pip install -r requirements.txt
```

That's it. No complex setup. Run it:

```bash
python main.py --listen 0.0.0.0 --port 8188
```

Now open `http://localhost:8188` in your browser.

## The Model: Z-Image-Turbo

I'm using **Z-Image-Turbo** — a distilled diffusion model that only needs **8 sampling steps** to produce high quality images. Compare that to 20-50 steps for standard SDXL.

On the 5090, a 1024×1024 image generates in **under 5 seconds**.

### Downloading from Hugging Face

You'll need three files from HuggingFace. Install the CLI first if you haven't:

```bash
pip install huggingface_hub
hf login   # paste your HF token (free account)
```

Then download each file:

```bash
# 1. The main diffusion model (UNET)
hf download Zhengyi/Z-Image-Turbo \
  z_image_turbo_bf16.safetensors \
  --local-dir ~/Downloads/z-image-turbo

# 2. The text encoder (CLIP - Qwen 3 4B)
hf download Zhengyi/Z-Image-Turbo \
  qwen_3_4b.safetensors \
  --local-dir ~/Downloads/z-image-turbo

# 3. The VAE
hf download Zhengyi/Z-Image-Turbo \
  ae.safetensors \
  --local-dir ~/Downloads/z-image-turbo
```

### Where to place the files

ComfyUI expects models in specific subdirectories:

```
ComfyUI/
└── models/
    ├── unet/
    │   └── z_image_turbo_bf16.safetensors   ← main model
    ├── clip/
    │   └── qwen_3_4b.safetensors            ← text encoder
    └── vae/
        └── ae.safetensors                   ← VAE
```

```bash
cp ~/Downloads/z-image-turbo/z_image_turbo_bf16.safetensors ComfyUI/models/unet/
cp ~/Downloads/z-image-turbo/qwen_3_4b.safetensors ComfyUI/models/clip/
cp ~/Downloads/z-image-turbo/ae.safetensors ComfyUI/models/vae/
```

## The Workflow

Z-Image-Turbo uses an **AuraFlow**-based pipeline. The node graph looks like this:

```
UNETLoader → ModelSamplingAuraFlow
CLIPLoader → CLIPTextEncode → ConditioningZeroOut
VAELoader  →
                              ↓
EmptySD3LatentImage → KSampler → VAEDecode → SaveImage
```

Key settings:
- **Steps:** 8 (sweet spot)
- **Sampler:** euler
- **Scheduler:** normal
- **CFG:** 1.0 (distilled models don't need high CFG)

## Automating It via API

ComfyUI's `/prompt` endpoint accepts the full workflow as JSON. Here's the basic flow:

```python
import requests, json, uuid

workflow = { ... }  # your node graph as dict

response = requests.post("http://localhost:8188/prompt", json={
    "prompt": workflow,
    "client_id": str(uuid.uuid4())
})

prompt_id = response.json()["prompt_id"]
```

Then poll `/history/{prompt_id}` until complete, and fetch the image from `/view`.

I use this to generate images directly from Telegram — I send a prompt, the GPU fires up, and the image comes back in seconds.

## Tips

**Prompt style:** Be descriptive. Z-Image-Turbo responds well to natural language descriptions — subject, style, lighting, mood all in one sentence.

**Resolution:** 1024×1024 is the sweet spot. Going larger (1536, 2048) works but takes more VRAM and time.

**Steps:** 8 is the default. Try 10-12 for slightly higher quality. Going above 12 gives diminishing returns with turbo models.

**Seed:** Fix your seed for reproducible results while iterating on prompts.

## On-Demand Loading (Don't Waste VRAM)

Running ComfyUI as a permanent service locks up ~20GB of VRAM constantly — even when you're not generating images. That's wasteful, especially if you're also running Ollama or other GPU workloads.

My approach: **start ComfyUI only when needed, stop it when done.**

I created two simple shell scripts:

```bash
# ~/scripts/comfyui-start.sh
#!/bin/bash
cd ~/ComfyUI
nohup python3 main.py --listen 127.0.0.1 --port 8188 > /tmp/comfyui.log 2>&1 &
echo $! > /tmp/comfyui.pid
echo "ComfyUI starting... check http://localhost:8188"
sleep 5  # give it time to load models
```

```bash
# ~/scripts/comfyui-stop.sh
#!/bin/bash
if [ -f /tmp/comfyui.pid ]; then
  kill $(cat /tmp/comfyui.pid) && rm /tmp/comfyui.pid
  echo "ComfyUI stopped."
else
  pkill -f "main.py" && echo "ComfyUI stopped."
fi
```

```bash
chmod +x ~/scripts/comfyui-start.sh ~/scripts/comfyui-stop.sh
```

I wired these into my AI assistant (OpenClaw) as a skill. When I say "generate an image", the skill automatically:

1. Checks if ComfyUI is running (`curl -s localhost:8188/system_stats`)
2. If not → runs `comfyui-start.sh` and waits for it to be ready
3. Submits the generation request
4. Returns the image
5. Optionally stops ComfyUI after a timeout if no more requests come in

This way the GPU is only loaded when actively generating. The rest of the time it's free for Ollama, coding agents, or whatever else needs it.

## Final Thoughts

Local image generation with ComfyUI on a proper GPU is genuinely better than cloud APIs for personal use. It's faster, cheaper (free after hardware cost), and completely private.

If you're in the UAE and running any kind of AI workload locally, happy to chat — drop me a message on [GitHub](https://github.com/balauae).
