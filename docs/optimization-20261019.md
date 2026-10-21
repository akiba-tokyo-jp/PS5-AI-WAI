# WAI generation optimization verification (2026-10-19)

Completed improvements 1–4 and console verification; adopted V16. First 256×256 decreased from 124.567 to 115.815 seconds, about 7.0% in these observations. Unloaded-model 1024×1024 decreased 184.026→161.397 seconds, about 12.3%. Same-input PNGs matched byte for byte. Loaded 1024 took 132.952 seconds after a size change and 117.600 for the next 1 matched request.

Aggregates 13 saved successes: 9 at 256×256 and 4 at 1024×1024, including old/new first runs. No console actions or additional generations for this write-up. [Extracted records](optimization-20261019-evidence.json) retain source SHA 256 without credentials, personal paths, prompt text or job IDs.

## Comparison conditions

WAI-Illustrious-SDXL v17.0,20 steps, CFG 5, Euler ancestral (`euler_a`), seed 20261018, manga LoRA 1, monochrome disabled, OpenPose 1. Staged comparisons use 256×256; size-switch tests 1024×1024. Same prompt/negative and size-matched reference control images. Saved conditions match except dimensions and per-upload control-image IDs.

PS5 CPU CLIP; PS5 Vulkan UNet/ControlNet/TAESD. No Windows inference. Initial comparisons reserved 9GiB Direct; V15/V16 used 9422MiB (about 9.201GiB). Each row is 1 observation, not a repeated mean.

First-run records have `model_loaded_before=false` and include loading. Loaded records use `true`; conditioning/weight-cache use is distinguished per measurement.

## 256×256 comparison of improvements 1–4

| Stage | State | PS5 generation | CLIP | UNet graph call | ControlNet | ControlNet evaluations |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| Baseline, first | Success |124.567 seconds |23.46 seconds |66.294 seconds |14.524 seconds |40 |
|① Allocation improvement, first | Success |124.026 seconds |23.43 seconds |66.210 seconds |14.487 seconds |40 |
|① Safe scratch reclamation, first | Success |124.148 seconds |23.48 seconds |66.326 seconds |14.493 seconds |40 |
|② Reduced duplicate CLIP, first | Success |113.543 seconds |12.70 seconds |66.496 seconds |14.531 seconds |40 |
|③ Added UNet improvements, first | Success |112.924 seconds |12.70 seconds |65.698 seconds |14.517 seconds |40 |
|④ Added ControlNet batching, first | Success |113.568 seconds |13.57 seconds |67.089 seconds |13.064 seconds |20 |
|④ Same configuration, loaded | Success |14.767 seconds |0.00 seconds |9.251 seconds |3.490 seconds |20 |
|① Reservation 9.201GiB, first (V15) | Success |112.885 seconds |13.54 seconds |66.418 seconds |13.051 seconds |20 |
|① CPU weight-group target 16MiB, first (V16) | Success |115.815 seconds |13.30 seconds |69.615 seconds |13.054 seconds |20 |

See [measurement records](optimization-20261019-evidence.json) for all rows. `generation_seconds` is internal PS5 generation time. Phase fields such as UNet graph calls are CPU wall time including waits, not isolated GPU compute. Nested columns cannot be summed to obtain total time.

### ① Memory allocation and reuse

Allocation and safe scratch-reclamation 256×256 outputs matched the baseline PNG. First times 124.026/124.148 versus 124.567 seconds differ only slightly; these 2 observations do not establish clear acceleration.

With model/OpenPose loaded, a CLIP-G conditioning-cache miss exceeded 9GiB CPU-weight reallocation capacity. Approximately 7.61GiB final Direct live plus 1.43GiB measured CPU CLIP weight peak requires sufficient total and contiguous free space.

**9422MiB (about 9.201GiB) reservation initialized successfully on hardware.**64MiB GPU tensor readback had 0 mismatches. V15 first 256×256 took 112.885 seconds with identical PNG. Later 1024×1024 could not allocate contiguous CLIP-G space despite approximately 1.593GiB total free: reservation alone did not solve fragmentation. API returned healthy with running/queued 0.

Then deployed V16 with 16MiB target CPU weight groups, preserving individual tensors, contents and ownership. V16 64MiB GPU readback had 0 mismatches; first 256×256 took 115.815 seconds with baseline identity. **1024×1024 then completed including conditioning recomputation.** Current adopted V16 includes 1–4; WAI weights/settings are unchanged. [Initialization and generation records](optimization-20261019-evidence.json)

Cut-cache release was reviewed: final-segment `prune` and per-compute end guards already release it; no duplicate release in `runner_end()` is needed. CPU CLIP layer streaming passed host checks but remains deferred.

## # ② Remove duplicate CPU CLIP computation

SDXL CLIP-G previously computed the hidden output selected by clip-skip separately from the final-layer pooled output. The shared encoder computation now runs in 1 graph, with the 2 distinct outputs retrieved separately.

CLIP decreased from 23.46 to 12.70 seconds, saving 10.76 seconds or about 45.9%. Total generation at this intermediate stage decreased from 124.567 to 113.543 seconds, about 8.9%. These are intermediate results for ② alone, distinct from the final V16 reduction of about 7.0%. Hidden and pooled outputs are not simply substituted for each other. PNG equality and 0 nonfinite values were verified. [Measurements](optimization-20261019-evidence.json)

## # ③ Share UNet computation and refine graph splitting

The common time/label embedding used by each ResBlock is retained at a cut to reduce repeated computation between partitions. The final partition boundary is merged only when measured workspace and retained-cache limits do not increase. Capacity checks, weight pinning, synchronization, and fault-stop conditions remain in place.

Compared with ②, total generation decreased from 113.543 to 112.924 seconds and UNet graph calls from 66.496 to 65.698 seconds. PNG equality and 0 nonfinite values were verified. The approximately 0.62-second difference comes from 1 observation per configuration; repeatability still needs verification. [Measurements](optimization-20261019-evidence.json)

## # ④ Batch ControlNet CFG

Positive and negative ControlNet computations now share a batch, reducing 40 executions to 20 over 20 steps. First-run ControlNet time decreased from 14.517 to 13.064 seconds, about 10.0%. Total generation changed from 112.924 to 113.568 seconds, so this comparison shows no additional overall reduction. PNG equality, 0 nonfinite values, and no CFG fallback were verified. [Measurements](optimization-20261019-evidence.json)

A warm generation with the same configuration took 14.767 seconds. However, CLIP took 0.00 seconds and UNet weight preparation about 0.069 seconds because cache state had changed since the first run. **The difference between 113.568 and 14.767 seconds must not be attributed solely to ControlNet batching**.

## First generation after app startup at 1024×1024

After normal app shutdown and restart, the same V16 executable and settings generated 1 image at 1024 with the model initially unloaded. Both the old baseline and V16 reported `model_loaded_before=false`. Requests were identical except for the control-image upload ID, and PNGs were byte-identical. [First-run measurements](optimization-20261019-evidence.json)

| Metric | Old baseline (V10) | Final configuration (V16) |
| --- | ---: | ---: |
| PS5 generation time | 184.026 s | 161.397 s |
| CLIP | 24.26 s | 14.10 s |
| UNet graph calls (including internal waits) | 80.811 s | 76.435 s |
| UNet weight preparation (including capacity checks) | 18.959 s | 18.851 s |
| ControlNet | 48.240 s / 41 calls | 42.337 s / 20 calls |
| CFG fallback | Present | None |
| Direct reservation | 9 GiB | Approximately 9.201 GiB |

Total generation decreased by **22.629 seconds, about 12.3%**. This compares complete configurations, including ControlNet CFG batching, fallback avoidance, and CLIP, UNet, and allocation changes; it does not isolate the effect of any single change. The user's model, steps, CFG, sampler, seed, LoRA, control conditions, and prompts were preserved.

This is 1 **first generation after app startup** per configuration. The PS5 OS file cache was not flushed; this does not establish identical fully cold storage conditions.

## Warm 1024×1024 verification with V16

After generating at 256×256, the same process generated 1 image at 1024×1024. The model was loaded, but CLIP was recomputed. The run completed a conditioning cache miss associated with the size change and the larger GPU computation. Only 1 additional image was then generated under identical conditions, using the conditioning cache. [1024 measurements](optimization-20261019-evidence.json)

| Metric | After resizing from 256 to 1024 | Next 1 image under identical conditions |
| --- | ---: | ---: |
| PS5 generation time | 132.952 s | 117.600 s |
| CLIP | 15.00 s | 0.00 s |
| UNet graph calls (including internal waits) | 76.814 s | 76.813 s |
| ControlNet | 32.386 s / 20 calls | 32.135 s / 20 calls |
| Peak Direct usage | Approximately 9.039 GiB | Approximately 9.039 GiB |
| Outstanding CPU CLIP shared Direct borrowing at completion | 0 | 0 |
| Nonfinite values / CFG fallback | 0 / None | 0 / None |

The 2 images were byte-identical to the 1024 baseline PNG and were retrieved through the OpenAI-compatible API. The actual image after resizing was also inspected. These 2 had `model_loaded_before=true`; no speed ratio against the old first-run 1024 time including loading is calculated. The difference between 132.952 and 117.600 seconds includes 15 seconds omitted by a CLIP conditioning cache hit. First-run improvements use the separate measurements in the preceding section.

## Output and memory verification

All 13 saved PNGs were read again, comparing their actual SHA 256 values with the recorded values. The 9 images at 256, all at 256 were all 256×256, 38,471 bytes, and matched the following SHA 256.

```text
b92fd8fba79b3d2cd3e76c4c1b218ecf1383b2d5de4431d66d852ad512f34a8a
```

The 4 images at 1024 were all 1024×1024, 505,949 bytes, and byte-identical across the old baseline, new first run, and warm runs. Their SHA 256 is:

```text
98eaa144fa64e8160352709df84f34420249d09023f03e108682d95de9dd8b0d
```

PNGs retrieved through the OpenAI-compatible and native APIs also matched. Every run had `nonfinite_values=0`, 0 outstanding CPU CLIP shared Direct borrowing, and no retained Direct fault. This establishes output equality and finite values within the inspected scope; PNG equality does not establish superior image quality. [Verification](optimization-20261019-evidence.json)

| Metric | First runs: baseline / ① / ② / ③ | First run: ④ | Warm run: ④ |
| --- | ---: | ---: | ---: |
| Direct reservation | 9 GiB | 9 GiB | 9 GiB |
| Direct usage at completion | Approximately 7.610 GiB | Approximately 7.610 GiB | Approximately 7.610 GiB |
| Peak Direct usage | Approximately 7.677 GiB | Approximately 7.695 GiB | Approximately 7.697 GiB |
| Maximum simultaneous CPU CLIP shared Direct borrowing during process lifetime | Approximately 1.430 GiB | Approximately 1.430 GiB | Approximately 1.430 GiB |
| Outstanding CPU CLIP shared Direct borrowing at completion | 0 | 0 | 0 |

V15 and V16 reserved approximately 9.201 GiB of Direct memory, used approximately 7.610 GiB at completion of the 256 generation, and peaked at approximately 7.695 GiB. Both had approximately 1.430 GiB maximum simultaneous CPU CLIP borrowing during process lifetime and 0 outstanding borrowing at completion.

Peak Direct usage includes memory borrowed by CPU CLIP. The CPU CLIP figure is the **maximum simultaneous borrowing during process lifetime**, not cumulative allocations. Warm rows retain the maximum from earlier generations. It was not sampled at the same time as completion usage, so do not add these figures as total VRAM. Process RSS can also overlap GPU mappings and is not CPU-only usage.

## Verification scope and remaining limitations

Verification completed for changes 1–4, V16 model-unloaded 1024 generation, resizing from 256 to 1024, a conditioning cache miss, and warm 1024 generation under identical conditions. Generation settings and model weights were preserved; PNG equality, 0 nonfinite values, and normal borrowing return were verified. Each condition succeeded 1 time; this is not a long-term stability guarantee.

- Long-running continuous catalog generation remains unverified.
- The user confirmed that TV colors were corrected. This is separate from the home-screen icon cache.
- The home-screen icon cache is a separate issue and is not included in completion of this performance verification.
- Additional mask and inpainting changes have not been deployed.
