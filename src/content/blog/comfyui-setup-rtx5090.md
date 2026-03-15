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

> **Note:** Z-Image-Turbo is a gated model. Before downloading, go to
> [huggingface.co/Zhengyi/Z-Image-Turbo](https://huggingface.co/Zhengyi/Z-Image-Turbo),
> log in with your HuggingFace account, and click **"Agree and access repository"** to accept
> the model license. The download will fail with a 403 error if you skip this step.

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
    ├── diffusion_models/
    │   └── z_image_turbo_bf16.safetensors   ← main model
    ├── text_encoders/
    │   └── qwen_3_4b.safetensors            ← text encoder
    └── vae/
        └── ae.safetensors                   ← VAE
```

```bash
cp ~/Downloads/z-image-turbo/z_image_turbo_bf16.safetensors ComfyUI/models/diffusion_models/
cp ~/Downloads/z-image-turbo/qwen_3_4b.safetensors ComfyUI/models/text_encoders/
cp ~/Downloads/z-image-turbo/ae.safetensors ComfyUI/models/vae/
```

## The Workflow

Z-Image-Turbo uses an **AuraFlow**-based pipeline. Here's the full node graph and what each component does:

```
┌─────────────────────────────────────────────────────────────┐
│  UNETLoader (node 28)                                       │
│    z_image_turbo_bf16.safetensors                           │
│         ↓                                                   │
│  ModelSamplingAuraFlow (node 11)   ← shift: 3              │
│         ↓                                                   │
│                    ┌──────────────────────────────────┐     │
│  CLIPLoader (30)   │  CLIPTextEncode (27)             │     │
│  qwen_3_4b         │    your prompt text              │     │
│  type: lumina2     │         ↓           ↓            │     │
│                    │  (positive)  ConditioningZeroOut │     │
│                    │                (negative / 33)   │     │
│                    └──────────────────────────────────┘     │
│                                                             │
│  EmptySD3LatentImage (13)  ← width, height, batch_size: 1  │
│                                                             │
│  VAELoader (29) ← ae.safetensors                           │
│                                                             │
│  All feeds into → KSampler (3) → VAEDecode (8) → SaveImage │
└─────────────────────────────────────────────────────────────┘
```

### What each node does

| Node | Role |
|------|------|
| **UNETLoader** | Loads the diffusion model (`z_image_turbo_bf16.safetensors`) |
| **ModelSamplingAuraFlow** | Wraps the model with AuraFlow sampling (shift=3 adjusts noise schedule) |
| **CLIPLoader** | Loads the text encoder (`qwen_3_4b.safetensors`, type: `lumina2`) |
| **CLIPTextEncode** | Encodes your prompt into conditioning vectors |
| **ConditioningZeroOut** | Creates empty (zeroed) negative conditioning — no negative prompt needed |
| **EmptySD3LatentImage** | Creates a blank latent canvas at your target resolution |
| **VAELoader** | Loads the VAE (`ae.safetensors`) for latent ↔ pixel conversion |
| **KSampler** | Runs the denoising loop — the actual generation |
| **VAEDecode** | Converts the output latent back into a pixel image |
| **SaveImage** | Saves the result to disk |

### KSampler settings

```
sampler:    res_multistep   (not euler — optimized for AuraFlow)
scheduler:  simple
steps:      8               (sweet spot for turbo)
cfg:        1.0             (distilled models ignore CFG guidance)
denoise:    1.0             (full generation from noise)
```

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

## Going Further — More Workflows & Models

Once you have the basics running, here's where to explore next:

### 🔍 Find Workflows

- **[ComfyUI Examples](https://comfyanonymous.github.io/ComfyUI_examples/)** — Official examples covering SDXL, ControlNet, inpainting, upscaling, video and more. Each comes with a downloadable workflow JSON.
- **[OpenArt Workflows](https://openart.ai/workflows)** — Community-submitted workflows. Browse by model or use case. Great for finding ready-to-use pipelines.
- **[CivitAI Workflows](https://civitai.com/)** — Models + matching workflows. Huge community library, especially for character and art styles.

### 🤖 More Models to Try

- **[FLUX.1](https://huggingface.co/black-forest-labs/FLUX.1-dev)** — State of the art photorealistic image generation. Heavier than Z-Image-Turbo but stunning quality. (Already have `flux2-dev.safetensors` locally!)
- **[SDXL Turbo](https://huggingface.co/stabilityai/sdxl-turbo)** — Fast SDXL distilled model, similar turbo approach.
- **[Stable Diffusion 3.5](https://huggingface.co/stabilityai/stable-diffusion-3.5-large)** — Latest from Stability AI.
- **[PixArt-Σ](https://huggingface.co/PixArt-alpha/PixArt-Sigma-XL-2-1024-MS)** — Efficient transformer-based model, great for artistic styles.

### 🎨 LoRAs (Style Add-ons)

LoRAs are small files that steer the model toward a specific style. Drop them in `models/loras/` and add a `LoraLoader` node between `UNETLoader` and `KSampler`.

- **[CivitAI LoRAs](https://civitai.com/models?type=LORA)** — Thousands of style/character LoRAs
- Already have `pixel_art_style_z_image_turbo.safetensors` locally — try it with Z-Image-Turbo!

### 📦 ComfyUI Manager

Install [ComfyUI Manager](https://github.com/ltdrdata/ComfyUI-Manager) to install custom nodes directly from the UI — no manual git cloning needed. Unlocks ControlNet, IP-Adapter, face detailers, and hundreds of community extensions.

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
```

Restart ComfyUI and you'll see a **Manager** button in the top menu.

---

## Final Thoughts

Local image generation with ComfyUI on a proper GPU is genuinely better than cloud APIs for personal use. It's faster, cheaper (free after hardware cost), and completely private.

If you're in the UAE and running any kind of AI workload locally, happy to chat — drop me a message on [GitHub](https://github.com/balauae).
