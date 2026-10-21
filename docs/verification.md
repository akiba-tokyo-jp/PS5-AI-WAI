# Hardware verification scope

See [logging verification](logging.md) for the latest log reduction, capacity limits, and speed comparison with identical PNGs.

2026-10-18 JST / PS5 FW 12.40. Original logs are private; this document publishes successful-generation measurements, hashes, and observation scope.
Full original logs, personal device photos, tokens, and failure videos are excluded.

For the latest performance/precision comparison, see [FP16/CFG verification](precision.md). The following records the previous version.

## Previous-version measurements

CPU CLIP / Vulkan UNet / Vulkan TAESDXL, Euler A, CFG 5, seed 135792. Runs used the same native process.
TAESDXL tile 256, overlap 0.5; batched CFG disabled; conditioning cache 2.

| Metric | 256×256 / 8 steps | 1024×1024 / 20 steps |
|---|---:|---:|
| Initial model loading | Included | Already loaded |
| PS5 generation time | 43.226907 s | 91.648546 s |
| Until official Windows SDK response | 44.515 s | 93.094 s |
| PNG size | 86,190 bytes | 857,419 bytes |

1024 PNG SHA256:

```text
3c4eaac8e2b5080a9b49c79c2fecbba213aac031438c3f8c064d010417c66986
```

The native 8188 PNG, OpenAI 8189 base 64 data, PowerShell HTTP retrieval, and README image were byte-identical.
Windows used the official OpenAI SDK3.34.0 and sent 1 POST with `max_retries=0`.
The earlier 20-step run producing the same adopted image took 87.133804 seconds; the latest 91.648546 seconds was 4.51 seconds (approximately 5.2%) longer.
This verified operation after log-capacity measures; it is not evidence of a speed improvement. The cause of the difference was not established from only 2 records.

## GPU evidence

Startup readback of a 64MiB F32 SCALE checked 16,777,216 values, finding 0 mismatches and the expected checksum.
Image generation sampled 1 each of completed UNet weight and Flash Attention operations after GPU synchronization.
Counter deltas for the 1024 generation were dispatch 138,826 / submit 29,518 / fence 29,518.

These do not measure GPU utilization, GPU-only execution time, or execution counts for every node.
They also do not prove readback across the entire 6GiB pool.

| Memory observation | bytes |
|---|---:|
| Direct data reservation | 6,442,450,944 |
| Owned data-pool peak during 1024 generation | 5,436,981,248 |
| Process peak RSS | 8,996,077,568 |

RSS and Direct usage can include shared memory; do not add them as total consumption.
CPU-only heap usage, PS5-wide VRAM utilization, and temperature were unavailable.

## Logs and recovery

The 3 high-frequency logs each retained a 4MiB current + 4MiB previous limit. Startup rotated 11 times in total; driver rotated 2 times.
All retained segments were at most 4MiB. After generation, engine available / GPU verified / model loaded were confirmed.
Queued and running jobs were 0. Existing services and the same console boot session were maintained.

Fatal-error failed/degraded notification and shutdown 409 were verified with host fault injection that performed no inference.
The hardware disk was not deliberately filled.

## Additional public-source verification

The public folder recompiled 11 native translation units and 1 RADV platform translation unit, linked the existing SHA-pinned dependency libraries, and verified SELF integrity valid.
This was a rebuild using a dependency cache, not an end-to-end clean build including the OS, SDK, and Mesa.
The original hardware-running eboot SHA is retained in [verification.json](verification.json).
There is no claim that the eboot produced by the public build was newly launched on hardware.
The public gateway PS5 ELF and Windows fixture were also built; 30 API-protocol checks and 7 logging/retained-state checks passed.
The public PowerShell client retrieved the actual PS5 model list and downloaded the saved 1024 PNG again, confirming the SHA above. No additional image was generated.

## Current limitations

- A same-condition PC benchmark, full-VAE quality, and output above 1024 remain unverified.
- Other firmware, other consoles, and app registration from an initial state have not been verified as a general procedure.
- Home-screen icon-cache refresh needs separate verification.
- Videos are retrospective recordings of actual results, not live inference or HDMI footage.
