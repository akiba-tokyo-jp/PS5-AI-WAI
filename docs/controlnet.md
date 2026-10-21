# ControlNet Canny / OpenPose console verification (2026-10-18)

See [2026-10-19 verification](optimization-20261019.md) for the current 9422MiB reservation, CPU CLIP grouping, ControlNet batching and 1024 results. The 9GiB configuration and times below are historical measurements.

On PS5 FW 12.40, connected Canny and OpenPose to WAI v17.0 and verified 256×256 /1024×1024 generation plus Windows PNG retrieval for both. 1 image per functional condition; not proof of long-run stability or completed quality optimization.

| Condition | Manga LoRA strength | PS5 generation | Windows API response |
|---|---:|---:|---:|
| canny 256 | 0 | 74.449 seconds | 75.672 seconds |
| canny 1024 | 1 | 162.557 seconds | 164.922 seconds |
| openpose 256 | 0 | 43.146 seconds | 43.531 seconds |
| openpose 1024 | 1 | 329.682 seconds | 333.125 seconds |

All cases: 20steps, CFG 5, Euler A, seed 20261018, control_strength 1. UNet, ControlNet and TAESDXL use Vulkan; CLIP uses CPU. FP32 and batched CFG were preserved. LoRA strength was 1 for 1024 and 0 for small tests. Mixed sizes, LoRA settings and initial model/shader states prevent deriving a speedup from this table.

Each image verified 20 finite UNet outputs, 40 GPU ControlNet evaluations, 10 residual tensors and 0 NaN/Inf values. Residuals reached the actual UNet. The official OpenAI SDK (max_retries 0) sent 1 POST to 8189;8188 on the same PS5 generated it. Returned and native PNGs matched byte for byte and passed visual review. Windows performed no model inference.

## Memory and fixes

Direct total 9GiB: 18×512MiB regions, Vulkan budget 8.75GiB. Expanded the old 256MiB limit per 1 region for the 464,230,404B workspace needed by 1024 UNet. Total reservation and GPU driver were unchanged. Startup GPU readback checked 64MiB, with 0 mismatches across 16,777,216 values; this is not a GPU test of all 9GiB.

CPU CLIP loads sequentially and borrows temporary weights from the existing Direct pool through CPU_Mapped buffers, including CPU LoRA parameters. Execution remains on CPU. No additional physical Direct allocation or Vulkan import of CPU weights is performed. Allocation/return counts and byte totals matched in every case; borrowed live bytes returned to 0 after generation.

`peak_gpu_bytes` sums reported owned-pool peaks, includes CPU borrowing and carries process-lifetime peaks forward. It is neither dedicated VRAM usage nor an independent per-job requirement. `peak_cpu_bytes` is process peak RSS and may include shared GPU mappings; do not add them. Direct reported size and available span are not current total free memory. GPU utilization and temperature were not measured.

Startup checks for saved inputs replaced a problematic console stdio route with bounded raw-FD reads. The same saved inputs were retrieved after restart with matching full SHA 256 values. The precise failing stdio operation remains unidentified.

## Inputs and quality scope

Provide a prepared edge map for Canny or pose map for OpenPose; automatic photo-to-OpenPose detection is unavailable. Match input/output dimensions. OpenPose 256 only checks composition: facial detail is coarse and the top edge is cropped.1024 does not guarantee exact joint alignment or final-artwork quality. Decoding uses TAESDXL, not full-VAE-equivalent quality.

Existing img2img / IP-Adapter paths remain, but earlier small-image quality issues are unresolved; this ControlNet test does not verify identity or quality. The masked-inpainting candidate was not yet deployed to the current app.

## Usage example

1. Upload a prepared PNG/JPEG map through authenticated `POST http://<PS5_IP>:8188/api/inputs`.
2. Pass the returned `image_id` as `control_image_id` to OpenAI-compatible `POST :8189/v1/images/generations`.
3. Set `controlnet` to `canny` or `openpose`; `control_strength` accepts 0.01–1.5 (default 1). These are WAI-specific extensions.

`scripts/Invoke-WaiControlNet.ps1` uploads, submits 1 generation and saves the PNG. After a timeout inspect health/native jobs instead of automatically retrying. Supply the token through a file.

## Models and licenses

Weights are not bundled. Both distributors' model cards state Apache-2.0. Source cards and full licenses are stored in `licenses/controlnet/`.

- [MIC-Lab Illustrious Canny](https://huggingface.co/MIC-Lab/illustriousXLv1.1_controlnet/tree/4fe9d46fadfad2ab46150749b89561b8cb998d38): install `illustriousXLv1.1_canny_fp16.safetensors` as `controlnet-illustrious-canny-f16.safetensors`. SHA 256 `3255c963a219a20040d2bef913216dd03d6da958068f5e69548dfe6e66528bcd`.
- [xinsir SDXL OpenPose](https://huggingface.co/xinsir/controlnet-openpose-sdxl-1.0/tree/23f966cd5cfdd3f7729c903e243d87152162d2b7): install `diffusion_pytorch_model.safetensors` as `controlnet-xinsir-openpose-sdxl-f16.safetensors`. SHA 256 `b8524e557a7df60d081f5d4a0eb109967d107df217943bf88c2d99b9ebcc06c5`.

Each is 2,502,139,104B. Full-file SHA 256 was also verified after PS5 transfer. Review separate WAI, manga LoRA and IP-Adapter terms.

## Phase timings (2026-10-18 record)

On 2026-10-19, [UNet/transfer/synchronization profiling and logging fixes](unet-profiling.md) reduced the same first OpenPose request from 194.54 to 184.03 seconds with identical PNGs. The following predates those changes.

ControlNet time is included in sampling time; do not count it twice.

Canny 1024 sampling took 132.95 seconds versus 296.24 for OpenPose 1024. CLIP took 24.52/24.51 seconds; total ControlNet evaluation 34.47/40.81; summed 6 recorded tensor loads 16.95/16.93. Most of the delay difference lies inside sampling outside ControlNet. UNet compute, transfer, synchronization and memory management were not individually separated; heat/logging is not a confirmed cause. Performance improvement is not considered complete.

### canny256

- get_learned_condition completed, taking 21.73s --- image.cpp:547
- sampling completed, taking 51.26s --- image.cpp:918
- decode_first_stage completed, taking 0.10s --- image.cpp:642
- generate_image completed in 73.69s --- image.cpp:1069

### canny1024

- get_learned_condition completed, taking 24.52s --- image.cpp:547
- sampling completed, taking 132.95s --- image.cpp:918
- decode_first_stage completed, taking 1.56s --- image.cpp:642
- generate_image completed in 162.12s --- image.cpp:1069

### openpose256

- get_learned_condition completed, taking 21.77s --- image.cpp:547
- sampling completed, taking 20.59s --- image.cpp:918
- decode_first_stage completed, taking 0.05s --- image.cpp:642
- generate_image completed in 42.56s --- image.cpp:1069

### openpose1024

- get_learned_condition completed, taking 24.51s --- image.cpp:547
- sampling completed, taking 296.24s --- image.cpp:918
- decode_first_stage completed, taking 4.82s --- image.cpp:642
- generate_image completed in 329.56s --- image.cpp:1069


Public-source rebuild and SELF integrity passed, with 24 initializer slots, 40 API checks and PowerShell client tests. The separate rebuilt public SELF was not redeployed to PS5. Machine-readable values are in `controlnet-verification.json`; console logs remain on a private D drive. Failure videos and secret tokens are not published.
