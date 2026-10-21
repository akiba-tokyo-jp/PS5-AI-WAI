# Normal-operation logging and performance

Verified normal mode on PS5 FW 12.40 on 2026-10-18 JST.
WAI-Illustrious-SDXL v17.0,768×1024,20steps, CFG 5, Euler A, seed 20261018, no negative prompt.

| Same-prompt condition | PS5 generation | Windows PNG retrieval |
|---|---:|---:|
| Before changes, slowed state | 417.90 seconds | — |
| Before changes, after 5-minute rest | 115.20 seconds | 116.61 seconds |
| After changes, initial model load included | 92.47 seconds | Not recorded |
| After changes, model loaded | **57.06 seconds** | **58.17 seconds** |

All cases produced the same PNG: 941,994 bytes, SHA 256:
`763b3eca54fda750cbc03081570dafb0d3127824abc0a834c70ae493175ba8cd`.
Warm conditioning cache hit and CLIP took 0.00 seconds. The old case spent approximately 7 seconds on CLIP, so
the entire time difference cannot be attributed to logging changes.
Sampling decreased 392.28→56.13 seconds; decoding 17.42→0.73 seconds.
The following 8 different-prompt images took 72.34–72.42 seconds on PS5,73.97–74.14 seconds to Windows retrieval.
These were resumed catalog jobs, not 8 additional test generations.
This does not establish completed endurance testing.

![Comparison using identical PNG](images/log-modes-same-768x1024.png)

## Additional 6 consecutive images

2026-10-18 14:26:38–14:33:53 JST: generated different prompts sequentially without pausing the original catalog.
The reported slowdown began at image 4 and persisted through image 6, so 6 consecutive images were checked.
Preserved 768×1024,20steps, CFG 5, Euler A, seed 20261018 and no negative prompt.

| Sequence | PS5 generation | Windows PNG retrieval |
|---:|---:|---:|
| 1 | 70.670 seconds | 72.36 seconds |
| 2 | 70.648 seconds | 72.23 seconds |
| 3 | 70.683 seconds | 72.77 seconds |
| 4 | 70.687 seconds | 72.77 seconds |
| 5 | 70.659 seconds | 72.62 seconds |
| 6 | 70.623 seconds | 71.95 seconds |

All 6 completed with GPU-use evidence and no errors. Each PNG saved by the OpenAI-compatible client
matched the native GET byte for byte and passed full PNG decoding.
Job gaps were 0–1 seconds; no additional generation POSTs, pauses, process restarts or PS5 setting changes.
Native API and gateway retained the same PID/started_at values.

Last 3/first 3 median-time ratio 0.9998; no 4–5× slowdown recurred in these 6 images.
Reported CPU peak 9,098,674,176 bytes and GPU peak 5,404,590,080 bytes also stayed constant across 6 images.
These are process/owned-pool lifetime peaks, not current usage or total VRAM measurements.
Unchanged peaks do not prove absence of leaks or completed long-term catalog operation.

[![Results from 6 consecutive images](images/log-modes-continuous-six.png)](media/log-modes-continuous-six.mp4)

[16-second Windows results recording](media/log-modes-continuous-six.mp4).
Recorded an actual Windows screen displaying completed results, not live generation.
Exact values and PNG hashes are in [logging-verification.json](logging-verification.json),
under `continuous_catalog_followup`.

## Normal and debug modes

Normal is the default. Add the following to select debug; this is an addition, not a complete configuration.

```json
{"logging":{"level":"debug"}}
```

Resolved 1 time before AGC initialization; switch through a normal app restart.

| Record | normal | debug |
|---|---|---|
| Successful per-submit/fence/SuspendPoint tracing | Omitted; counters retained | Detailed |
| Normal allocation/free/internal cleanup | Omitted; usage/peak retained | Detailed |
| Start/completion, phase time, model state, GPU evidence | Retained | Retained |
| Warnings/errors/abnormal returns/retention/critical completion | Retained; immediate sync | Retained; immediate sync |
| SD.cpp DEBUG/VERBOSE and split full prompts | Omitted | Detailed |
| Graph data/scratch plans | Retained for current diagnostics | Retained |
| Duplicate raw stderr files | Not created | Not created |

Normal mode still checks actual GPU returns, fences/quiescence, memory plans and finite values.
Suppressing logs never converts GPU failures into successes.
The warm run omitted 350,510 records before formatting/writing.
Measured startup/driver fsync totaled 24 calls and approximately 36ms waiting.
An earlier slowed log at the same size/steps had approximately 5,500 calls and hundreds of seconds waiting.
Counters exclude other I/O such as rotation; cumulative times on separate threads are not wall-clock totals.

Normal mode retained 300 graph plans; engine logging totaled 316 events/192,710 bytes per image.
Routine debug logs also batch synchronization at 1,024 records /512KiB /1 elapsed second.
The 1-second threshold is checked on the next record, not an idle timer guarantee.
Critical records, failures and final checkpoints synchronize immediately.

## Log retention limits

| Continuous log | 1 segment | Generations | Total |
|---|---:|---:|---:|
| startup / driver / pool |4MiB each |2 each |24MiB |
| engine.jsonl | 16MiB | 2 | 32MiB |
| Native API api.log |2MiB |2 |4MiB |
| OpenAI gateway api.log |2MiB |2 |4MiB |
| **Primary continuous logs** | | | **64MiB** |

Download-side logs occupy 24MiB; remaining logs are on the data side. Primary log retention does not grow indefinitely with generation.
Small per-startup bootstrap/witness files and one-use development receipts are separate.
Generated images/job histories are also separate and are not automatically deleted.
Retrieve migrated previous files separately if old logs already exceeded their bounds.
All measured console files remained within their limits.

GPU computation, submit/fence and ownership implementations, model, FP32 and 6GiB Direct pool were preserved.
Verified 2 startup precision readbacks, 4 convolution cases, 40 UNet output checks and 0 NaN/Inf values.
Matching-image counters also remained 137,630 dispatches /29,143 submissions /29,143 fences.
These are not GPU-utilization measurements.

The console-executed binary SHA 256 is
`a1ec807794ab605e7c814cc12328ce54208bcaa1f68af2dd05a8400e21f51807`.
The separate public-source build passed SELF integrity and is distinguished from the executed artifact.
Records: [logging-verification.json](logging-verification.json); source correspondence: [source-snapshot.json](source-snapshot.json).

## 2026-10-19: Direct pool and normal-release synchronization boundaries

Fixed normal Direct-release phase names missing from the normal-mode allowlist with ControlNet. Retained every checked record while batching successful pool allocation/import/return sync at 128 records/64KiB/1 second (checked on the next record), with a mandatory checkpoint before PNG publication. Errors, debug, initialization/teardown and physical allocation/mapping still sync immediately. Current/previous limits remain. See [phase measurements and regression](unet-profiling.md).
