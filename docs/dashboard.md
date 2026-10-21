# CPU-rendered status screen

Navy background, mint/lavender accents, smooth text and status, memory and log cards.
Updated to the user-provided character on 2026-10-18:512px home PNG and 72px in-app image files. A2026-10-19 photo still showed the old home icon; its stale cache identifier was confirmed by read-only inspection. Home display refresh remains incomplete. Text/cards use CPU rendering, preserving 32MiB display Direct buffers and roughly 1-second refresh.
No Vulkan drawing context or external GUI library was added.

![Synthetic dashboard preview](images/dashboard-colour-preview.png)

The same rendering code runs on Windows with synthetic data; this is not PS5 HDMI footage.
PS5 APIs verify completed presentation and heartbeat updates. TV colors and text clipping require separate visual checks.

## Displayed information

- App/model names, PS5 UTC clock and display-loop heartbeat.
- Processing state, job and progress when valid measurements are available.
- Usage, capacity and peak of the application's reserved Direct data pool, not all VRAM.
- Memory values with identified sources and completed-fence counters.
- Recent filtered logs; lines containing credential labels are omitted.

Unavailable values are omitted together with their labels. The current native publisher provides no CPU/GPU utilization, temperature or current RSS.
Preview RSS and driver-arena values are synthetic examples of display capabilities.
Samples older than 3 seconds invalidate status/metrics. Display heartbeat is not inference progress.

## API and compatibility

Authenticated native `GET /health` includes `tv_display` when available.

```json
{"tv_display":{"ready":true,"heartbeat":128,"last_presented_frame":128,"fault":false,"stopped":false}}
```

Values illustrate the response format. `ready` and `last_presented_frame` require positive VideoOut presentation completion.
If its mutex is busy, `tv_display` is null and the API does not wait.
The CPU gateway skips additional upstream health metadata within depth/capacity bounds.
User generation requests and upstream job JSON retain their strict flat format.

`gpu_verified` means UNet GPU computation completed during image generation, so it remains false until the first image after startup.
Distinguish completed display GPU checks, model-loaded state and completed image GPU computation.

## Verification

Verified 10 existing display-data cases, frame-buffer guards, 3 health states, 10 upstream metadata-parser cases and 30 existing API tests.
Host tests cover metadata/rendering only, without image inference.
The new console app generated the same 768×1024/20steps case: first 92.443825 seconds, Windows retrieval 94.25 seconds.
The old display's first run took 92.473289 seconds under comparable conditions; generated PNGs matched byte for byte.

PNG SHA256: `763b3eca54fda750cbc03081570dafb0d3127824abc0a834c70ae493175ba8cd`.
Preserved inference engine, GPU driver, FP32,6GiB Direct data pool, model and sampler.
Executed console SELF SHA 256: `be1067a7520dce661129a2a1cab8c85fd92bd01be17d1d82b91917199679fac6`.
The separate public-source rebuild is distinguished from the executed console artifact.

The font is a static OFL-1.1 bitmap; its [font license](../licenses/WAI-Dashboard-Glyphs-OFL.txt) is retained.

## Continued-generation measurements

Continued the original catalog and observed 6 naturally consecutive images. No extra warm-generation POST was submitted.

| Catalog ID | PS5 generation time | Windows PNG retrieval |
|---|---:|---:|
| horror-urban | 68.854 seconds | 70.31 seconds |
| horror-yokai | 68.845 seconds | 70.55 seconds |
| horror-panic | 68.835 seconds | 70.72 seconds |
| essay-diary | 68.794 seconds | 70.09 seconds |
| essay-parenting | 70.620 seconds | 72.26 seconds |
| essay-travel | 70.590 seconds | 71.95 seconds |

All 6 passed GPU-use checks, full PNG decoding and byte identity between OpenAI-compatible saved files and native GET.
Display heartbeat advanced 424→1062; positive VideoOut completion was observed, with 0 faults.
Native/gateway processes were unchanged. Temperature/utilization were unavailable; this does not prove full catalog completion or absence of memory leaks.
Machine-readable data: [dashboard-verification.json](dashboard-verification.json).

Reproduce host checks with `python tests/dashboard.py --config .local/build.json --out <private-build-dir>`.
Checks frame guards and 10 cases of additional upstream health metadata. No host image inference.

## Windows success recording

[![Completed-results verification screen](images/dashboard-success.png)](media/dashboard-success.mp4)

[16-second Windows recording](media/dashboard-success.mp4), showing completed console results; not live generation or PS5 HDMI capture.
H264,1280×720,192 frames. Full decoding and visual frame inspection passed.

## Color-format fix on 2026-10-19

Matched CPU BGRA byte order to VideoOut BGRA SDR. Only 1 display-attribute constant changed. Verified completed presentation, advancing heartbeat, 0 faults, GPU readback and OpenPose 1024 generation. TV colors still awaited user confirmation at this point. See [profiling and display fix](unet-profiling.md).
