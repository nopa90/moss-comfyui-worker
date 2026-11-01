# moss-comfyui-worker

A [Runpod serverless](https://docs.runpod.io/serverless/overview) worker image that can
actually run **MOSS-Image edit workflows**, built by GitHub Actions and pushed to Docker Hub.

## Why this repo exists

`runpod-workers/worker-comfyui` is the standard "run any ComfyUI workflow as an API" worker:
the workflow graph rides the job payload, so a custom workflow needs no custom image.

The problem is the pinned version. In worker-comfyui **5.11.0** (2026-09-21):

```
ARG COMFYUI_VERSION=0.34.0
```

MOSS-Image core support landed in **0.42.0**, and `TextEncodeMOSSImageEdit` (the node that
takes reference images through `images.image_0 … images.image_N` with the VAE connected)
landed in **0.43.0**. On the stock image, a MOSS workflow fails node validation — it does not
silently misrender.

So this repo builds *upstream's own Dockerfile* with exactly one value changed. Nothing else
is forked, so upstream fixes, dependency pins and the build-time
`python main.py --quick-test-for-ci --cpu` smoke test all still apply.

## What gets built

| | |
|---|---|
| Base | `nvidia/cuda:12.8.1-cudnn-runtime-ubuntu24.04` |
| Upstream worker | `runpod-workers/worker-comfyui` @ `5.11.0` (override with `worker_tag`) |
| ComfyUI | **0.43.0** (override with `comfyui_version`) |
| Image | `<DOCKERHUB_USERNAME>/moss-comfyui:0.43.0`, `:latest`, `:<sha>` |
| Custom nodes | none — MOSS support is core |

The `verify` job pulls the pushed image and asserts, inside it:

- `import comfy_extras.nodes_moss_image` succeeds and `TextEncodeMOSSImageEdit` exists;
- `moss_image` is a recognised `CLIPLoader` type.

A green run therefore proves the image can load the graph, not just that Docker exited 0.

## Repo secrets required

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub account (same one used by `voxcpm2-runpod`) |
| `DOCKERHUB_TOKEN` | Docker Hub access token with read/write |

```bash
gh secret set DOCKERHUB_USERNAME --body '<username>'
gh secret set DOCKERHUB_TOKEN    --body '<token>'
```

## Setup, once

```bash
gh auth login
gh repo create moss-comfyui-worker --public --source=. --remote=origin --push
gh secret set DOCKERHUB_USERNAME --body '<username>'
gh secret set DOCKERHUB_TOKEN --body '<token>'
gh workflow run build.yml            # or just push to main
gh run watch
```

Public repo so Actions minutes are unmetered — the CUDA + torch build is heavy and the
image is public on Docker Hub anyway.

## Weight layout on the network volume

The worker's `src/extra_model_paths.yaml` maps a network volume into ComfyUI under the
**older** folder names. Anything outside this map is invisible to the loaders:

```yaml
base_path: /runpod-volume
checkpoints, clip, clip_vision, configs, controlnet, embeddings,
loras, upscale_models, vae, unet      # <- no diffusion_models, no text_encoders
```

Staged layout for the int8 lane (15.71 GB, sizes verified byte-exact against
`Comfy-Org/MOSS-Image`):

```
/runpod-volume/models/
├── unet/moss-image/moss_image_sft_int8_convrot.safetensors    5,413,698,958
├── clip/moss-image/qwen3.5_9b_int8_convrot.safetensors        9,748,121,532
├── loras/moss-image/moss_image_edit_lora_rank_64_bf16.safetensors  297,382,216
└── vae/moss-image/flux2-vae_small.safetensors                   249,521,340
```

Because the subdirectory is preserved, workflow strings stay
`moss-image/<file>` — same as the local tree in `~/Projects/AI-Studio/models/comfyui/`,
only the category folder differs.

## Job payload

```json
{
  "input": {
    "workflow": { "<node-id>": { "class_type": "...", "inputs": {} } },
    "images": [
      { "name": "ref-a.png", "image": "<base64>" },
      { "name": "ref-b.png", "image": "<base64>" }
    ]
  }
}
```

- `workflow` must be **API format** (`Workflow → Export (API)`), not the UI export.
- Every `LoadImage` node must name an entry in `images`.
- Results come back base64 in `output.images[0].data`; they are deleted after 30 minutes.
- Use `/run` + poll `/status/<id>` — a cold start does not fit in `runsync`'s hold.

The payload for the 2-reference MOSS edit is generated locally by
`~/Projects/AI-Studio/scripts/build-moss-serverless-payload.py` (it also swaps the bf16
loaders for the staged int8 files).

## Adding custom nodes later

MOSS core needs none, but the `rapid-moss` (4/8-step) and `armV5` LoRA lanes do. They cannot
come from a volume — bake them in a derived Dockerfile:

```dockerfile
FROM <DOCKERHUB_USERNAME>/moss-comfyui:0.43.0
RUN comfy-node-install <node-id-or-git-url>
```

and point the workflow's node `class_type`s at what that node registers.
