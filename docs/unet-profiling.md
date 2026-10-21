# UNet phase timing, log synchronization, and display-color fixes (2026-10-19)

Both runs used the same PS5 on FW 12.40, WAI v17.0, OpenPose, 1024×1024, 20steps, CFG 5, Euler A, seed 20261018, manga LoRA 1, no automatic monochrome, and identical prompts, negatives, and input maps. Each run followed normal app shutdown/startup with the model initially unloaded. Reservation 9GiB, 18 regions of 512MiB, budget 8.75GiB, RADV, and FP32 arithmetic were preserved.

| Comparison | PS5 generation time | Model state |
|---|---:|---|
| v8: instrumentation only, before synchronization changes | 194.543737 s | First run |
| v10: log synchronization + BGRA display-format fix | 184.026332 s | First run |

This saved approximately 10.52 seconds or 5.4%. Each configuration generated 1 image; this is not a long-term speed guarantee or statistical comparison. The v10 Windows/OpenAI response took 184.906 seconds. Windows performed no inference.

Verification confirmed 20 sampler steps; Vulkan paths for UNet, ControlNet, and TAESDXL; NaN/Inf 0; 0 outstanding CPU CLIP temporary borrowing; and byte-identical OpenAI/native PNGs. PNGs also matched the earlier v7 and v8 outputs exactly.

PNG SHA 256: `98eaa144fa64e8160352709df84f34420249d09023f03e108682d95de9dd8b0d`. The first CFG batch failed 1 time; only that 1 step used a fallback computing conditions individually on the GPU. This was not CPU UNet inference. Recorded counts were 21 successful UNet outputs, graph_total 22 including the failure, and 41 ControlNet calls. This does not claim full batching of every step.

## Breakdown observed on CPU

These are CPU-side elapsed times measured with CLOCK_MONOTONIC around calls. **They are not GPU timestamps.** graph_call includes internal GPU waits; weight_prepare includes loading, placement, and capacity adjustment; workspace includes memory management and its logging. graph_total and cleanup have nested relationships, so do not sum columns and compare them with total generation time.

| Measured interval | v8 seconds | v10 seconds |
|---|---:|---:|
| `unet.graph_total` | 115.336936 | 105.421987 |
| `unet.graph_call_including_internal_wait` | 79.412522 | 80.810967 |
| `unet.weight_prepare_including_capacity` | 19.969031 | 18.958531 |
| `unet.workspace_prepare_allocate` | 9.058852 | 1.718110 |
| `unet.activation_upload` | 0.365722 | 0.366147 |
| `unet.output_capture_readback` | 2.464205 | 0.899616 |
| `unet.explicit_workspace_wait` | 0.013990 | 0.016127 |
| `controlnet.graph_total` | 47.803109 | 47.345404 |
| `Direct_pool_log_fsync` | 7.485292 | 0.183589 |
| `startup_log_fsync` | 5.246403 | 0.147166 |
| `engine_structured_log` | 2.263281 | 2.284576 |

UNet graph_call sums multiple graph partitions. activation_upload covers observed input-transfer calls, not all PCIe/memory traffic. Observed weight copying also covers only part of the work. Small explicit workspace waits do not establish absence of internal waits.

The backend idle polling added in v10, excluding logging, took 0.124910 seconds over 124476 calls. Quiescence calls including logging took 0.478794 seconds. Both are CPU elapsed times at the measured sites, not total GPU synchronization time. All measurements had 0 invalid clocks.

## Corrected log synchronization

The normal Direct-buffer-free phase `native_Direct_owned_buffer_free` did not match the existing normal-phase list, causing immediate fsync during normal work. Normal import records also synchronized immediately. Earlier v7 observations recorded 3588 startup fsync calls for both Canny and OpenPose, taking 5.971878 and 77.135470 seconds respectively. The 71.16-second difference directly explains part of the delay, but does not establish the cause of the entire generation-time difference, temperature, or clocks.

v10 batches synchronization while retaining checked records for normal work. Startup uses existing normal boundaries; the pool synchronizes at 128 records / 64KiB / 1 second, evaluated on the next write. A mandatory pool checkpoint was added before PNG publication. Debug, initialization, shutdown, physical allocation/mapping, and fault synchronization remain immediate. GPU state queries, fences, ownership checks, and return checks are retained.

- Pool fsync: 5016→98 calls, 7.485292→0.183589 seconds.
- Startup fsync: 3597→98 calls, 5.246403→0.147166 seconds.
- GPU quiescence queries remained at 124476 in both runs. Successful records were not reduced. Existing limits, including current + previous logs of 4MiB each, were retained.

v8 did not reproduce the earlier 77-second startup fsync. Therefore, the entire difference between the earlier 329.68 seconds and this run's 184.03 seconds is not attributed to this fix. CPU temperature and GPU clocks were not measured. UNet graph_call remains the largest observed interval; separating GPU computation from internal waits remains future work.

## Measured contiguous-space shortage

A diagnostic inspecting the bitmap on pool allocation failure recorded a request of 167772160B (160MiB), total free space of 799571968B (762.53MiB), largest contiguous free span of 159383552B (152MiB), and region limit 512MiB in the same job. This directly measured fragmentation: insufficient contiguous space despite enough total free space. It is not proof that the total reservation was simply too small.

The first CFG fallback occurred in the same job, but a direct causal relationship with allocation failure was not established. Reviewing placement order for large work buffers and persistent weights is a next candidate. This version did not change allocator placement, reservation size, or CPU/GPU arithmetic. The 8.502GiB pool peak is the owned-pool process peak including CPU temporary borrowing, not dedicated GPU VRAM usage. Peak RSS may also contain shared mappings and must not be added.

## App display colors and icon

The user's photo showed reddish-purple cards and a discolored character. CPU writes used BGRA order, while VideoOut used another format, `0x8000000022000000`. Only 1 display attribute was changed to `0x8000000000000000`, the [measured BGRA SDR format in the reference implementation](https://github.com/mihawk-99/PS5_Vulkan/blob/f410e67a0241b3e8edcdbcd2a9b285dfc4a412ed/docs/HARDWARE_FINDINGS.md#L54). Display memory 32MiB, drawing, flip, lifetime, and approximately 1-second updates were preserved.

CPU checks verified R/G/B byte order, tiled/linear equality for all 2073600 pixels, and boundaries. Hardware confirmed completed display updates, a continuing heartbeat, and 0 display faults. TV color confirmation was pending the user.

![Mock-data preview using the same drawing code](images/dashboard-colour-preview.png)

This is a CPU preview on Windows, not a PS5 HDMI capture.

The user's photo confirmed that the home screen still showed the old icon. All 4 deployed PNGs matched the new character's SHA, but registered icon 0Info retained an old cache identifier. Updating image files and refreshing the home display are separate. Investigation remained read-only: no direct DB editing, forced ShellUI termination, or console restart. Home-icon refresh is not claimed as successful.

## Retained evidence and recovery

The hardware-used SELF SHA 256 was `31e28c339ad0cc42691478253e083bf4c456c3f9ebca5ba7b84e82e694a44b2d`. The public-source rebuild is a separate artifact; this does not mean that rebuilt SELF ran on hardware.

The previous SELF and configuration were retained. Before testing, the catalog remained in STOP state after finishing its current 1 image. PID, start time, boot, and app ID were checked; normal OS app shutdown preceded replacement. GPU 64MiB readback verified 16777216 values with mismatch 0 before generation. No forced process termination, manual GPU release, or firmware update was performed.

Normal recovery waits for the current job to finish and the app to become idle, shuts down through the OS, restores the saved SELF, and starts normally. No kill based solely on PID or duplicate launch is used. The catalog remained stopped at the end of investigation.

Windows host regressions verified retention of 200 checked records; 128-record / 64KiB / 1-second boundaries; immediate synchronization for debug/initialization; the final checkpoint; and sticky faults on synchronization failure. They performed no model inference or PS5 connection. Timing helpers also verified interval identity, nested scopes, errno/return-value preservation, and no measurement when disabled.

```powershell
python tests/direct_logsync_host.py --compiler <mingw-clang.exe> --out <host-test-output>
# phase_profile_host.cpp includes native/app and is built for Windows with -UNDEBUG -lwinpthread.
```

[![Completion segment of the actual Windows recording](images/controlnet-openpose1024-logsync.png)](media/controlnet-openpose1024-logsync.mp4)

The video is the final 12 seconds of an actual Windows API-operation recording. PS5 generation took 184.03 seconds; video duration is not generation time. It is not PS5 HDMI footage. Public-source rebuild, SELF consistency, ordering of 24 constructors and matching external imports, and Windows host regressions passed. The rebuilt public SELF was not rerun on hardware.
