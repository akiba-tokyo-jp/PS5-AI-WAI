# Hardware comparison of FP16 and CFG batching

PS5 FW 12.40, WAI v17.0, 20 steps, Euler A, CFG 5, seed 135792. The official Windows OpenAI SDK called the PS5 API and retrieved actual PNGs. Inference ran on PS5.

## Adopted configuration

**FP32 arithmetic + batched CFG** was adopted. Conditional and unconditional inputs are processed together as batch 2, reducing UNet calls from 40 to 20 over 20 steps. Model, steps, seed, and decoder settings were preserved.

| Warm repeats under identical conditions | FP32 sequential CFG | FP16 sequential CFG | FP32 batched CFG |
|---|---:|---:|---:|
| 256×256 / 20 | 35.44 s | 41.88 s | **14.62 s** |
| 1024×1024 / 20 | 84.09 s | Not measured | **79.27 s** |

Batching reduced time by 58.7% at 256 and 5.7% at 1024. The 256 PNG was byte-identical to the sequential version; the 1024 equality result was `true`.
Each repeat had 2 conditioning embedding cache hits and 0 model reloads. These are single observations, not guaranteed speeds.

The first 256 request, including model loading, took 57.30 seconds for FP32 sequential, 56.97 seconds for FP16 sequential, and 48.18 seconds for FP32 batched CFG.
The first 1024 request, with the model loaded but a conditioning cache miss, took 91.65 seconds for FP32 sequential, 322.82 seconds for FP16 sequential, and 82.80 seconds for FP32 batched CFG.
First runs and repeats are not mixed to calculate speed ratios. The FP16 1024 repeat was not measured.

## FP16 verification

On hardware, `shaderFloat16`, `storageBuffer16BitAccess`, and `uniformAndStorageBuffer16BitAccess` were all true.
FP16 startup removed the variable itself with `unsetenv("GGML_VK_DISABLE_F16")`; actual GGML logs confirmed `fp16: 1` and variable absence. Setting `=0` was not used.
The PS5 native startup code performs this change. Changing a Windows environment variable alone does not switch PS5 behavior.

F16 weight matrix operations and F16 K/V Attention were read back from the actual GPU and checked. Direct convolution was separately compared across 2 shapes with CPU double-precision reference values derived from the actual rounded inputs. Maximum absolute error was approximately 4.71e-7, with 0 mismatches.
Direct convolution is enabled only after this verification, avoiding a large im 2col intermediate tensor. The FP16 version generated a 1024 PNG, but no speed benefit was verified in this configuration; FP32 was restored as default.

## Image quality and memory

![Hardware output with FP32 batched CFG](images/wai-cfg-batch-1024.png)

The [actual FP16 image with the same prompt and seed](images/wai-fp16-1024.png) is also retained. Visual inspection showed the overall manga portrait intact, without obvious missing content or solid-color output. Lines and colors differ; this is not a test of full perceptual equivalence or stability across all prompts.

For FP16 at 1024, 99 outputs, including 40 UNet outputs, were checked: 12,888,576 values with 0 NaN/Inf.
For batched CFG at 1024, 20 UNet outputs were checked with 0 NaN/Inf. This does not mean every internal tensor was inspected.
Every run verified completed GPU UNet/Flash witnesses and a ready state after generation. Witnesses are sampled observations, not total GPU operation counts or utilization.

The Direct reservation remained 6 GiB. Peaks at 1024 were 5,436,981,248 bytes for FP32 sequential, 5,436,981,248 bytes for FP16 sequential, and 5,869,420,544 bytes for FP32 batched CFG.
Weights were already FP16; intermediate outputs and accumulation still include F32. Enabling FP16 arithmetic does not halve memory usage.
RSS may include shared mappings and must not be added to Direct usage. Total VRAM utilization and temperature were not measured.

## Reproduction code and evidence

`tools/build.py --precision fp32` / `--precision fp16` builds comparison versions from the same public source. The default is fp 32. Batching is configured by `engine.batched_cfg`.
The public-folder rebuild and SELF inspection passed, along with 48 scratch-admission checks and NaN/Inf injection into the actual finite-check function. Host tests do not perform model inference.
This was a rebuild using a dependency cache. Building all dependencies from a clean environment and first registration remain unverified.

[Machine-readable measurements](precision-verification.json) / [Public-source rebuild verification](precision-build-verification.json) / [Short successful-results video](media/precision-cfg-verified.mp4)

The video is a retrospective recording of the Windows screen displaying actual images and measured logs. It is not live generation or HDMI footage.
Existing FTP/Telnet/WebSrv, firmware, and system areas were preserved; the PS5 was not restarted during verification. Original logs remain private.
