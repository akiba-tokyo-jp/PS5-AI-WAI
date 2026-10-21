Current version: manga LoRA strength 1 is enabled by default; disable it with `manga_strength:0`. Automatic monochrome defaults to false. OpenAI JSON accepts both fields as WAI extensions. Canny/OpenPose ControlNet was verified at 256/1024: [Results and input preparation](controlnet.md).

# API reference

See [Logging configuration](logging.md) for normal/debug levels and storage limits. OpenAI endpoint behavior is preserved.

Both services require authentication tokens. Tokens are not embedded in source: each process reads a dedicated startup file, removes it and retains the value in memory.
Connections are restricted to the same private IPv4 /24. This is HTTP; do not forward or expose its ports outside the LAN.

## OpenAI Images-compatible subset: 8189

Client base URL: `http://<PS5_IP>:8189/v1`; model: `wai-illustrious-sdxl-v17.0`.
A separate CPU process sends exactly 1 POST to 8188 on the same PS5, waits for completion and returns the actual PNG.
Inference is not forwarded to Windows or a cloud service.

| Method and path | Behavior |
|---|---|
| GET `/v1/models` | List models |
| GET `/v1/models/wai-illustrious-sdxl-v17.0` | Model information |
| POST `/v1/images/generations` | Synchronous response: base 64 PNG or signed URL |
| POST `/v1/images/edits` | Multipart image and optional mask; see [mask support](mask-api.md) |
| GET `/health` | Gateway and same-PS5 backend status |
| GET `/openapi.json` | OpenAPI 3.1 document |
| GET `/v1/images/{id}.png?expires=…&signature=…` | Returned signed image URL |

Normal routes use `Authorization: Bearer <token>`. Never put the API secret token in a URL.
Image signatures cover 1 image, at most 1 hour and the current server session. Treat the URL itself as image-access authorization.
Signed image URLs retain LAN restrictions; modified or expired signatures return 401.

| Parameter | Support |
|---|---|
| prompt | Required; at most 4096 UTF-8 bytes |
| model | Only the model above |
| n / output_format / stream | Only 1 / png / false |
| size | auto (1024×1024), or WIDTHxHEIGHT with each side 64–1024 in multiples of 64 |
| response_format | b64_json (default) / url |
| quality / background | auto or standard / auto or opaque |
| user / safety_identifier | Metadata up to 256 characters; does not add moderation |
| negative_prompt | Extension; at most 4096 UTF-8 bytes |
| steps / cfg_scale | Extensions: 1–100 (default 8) /0–30 (default 5) |
| seed | Extension: -1 or a nonnegative signed 64-bit integer |
| sampler | Extension: euler / euler_a (default) / dpm++2m |

Optional null values use defaults. The generations JSON body limit is 24KiB. See [mask API](mask-api.md) for multipart limits.
Unsupported parameters, batches, transparent output, JPEG/WebP and streaming are rejected with 400 or another error. See the [masked-edit specification](mask-api.md).
Success returns `{"created":...,"data":[{"b64_json":"..."}]}` or `url`.
`x-request-id` identifies the adapter request; `X-PS5-Native-Job-ID` identifies the actual native generation.
Failures use OpenAI-style `error.message/type/param/code` fields.
Only 1 synchronous generation is accepted; contention returns 429. Health remains responsive while awaiting the result. The wait limit is 30 minutes.

## Native asynchronous API: 8188

| Method and path | Behavior |
|---|---|
| GET `/health` | model_loaded, GPU status, backend and running/queued counts |
| POST `/api/generate` | prompt / negative_prompt / width / height / steps / cfg_scale / seed / sampler; returns 202 and job_id |
| GET `/api/jobs/{job_id}` | queued / running / completed / failed, progress, error and image URL |
| GET `/api/images/{image_id}` | Retrieve the actual PNG |

An HTTP timeout or disconnect does not cancel native generation.
If the ID is known, inspect that job. Never automatically resend a POST whose outcome is unknown.
In a console test that closed the connection during generation, the 1 POST completed and its PNG was recovered through a separate route. See [verified scope](ip-lifecycle-20261019.md#recovering-results-after-a-disconnect).
If a normal synchronous OpenAI client disconnects after acceptance but before receiving an ID, the current API cannot reliably discover its unknown job. The console recovery test used operator access to internal correspondence logs to identify it. Client correlation IDs and idempotency are future improvements.
See [Disconnect, 6 consecutive jobs and normal-restart verification](operations-recovery-20261019.txt).
Fatal retained states report failed/degraded; new generation returns 503, and shutdown that cannot wait safely returns 409.
Administrative shutdown does not itself prove normal application cleanup. See [Operation](deployment.md).

## Reference-image and ControlNet extensions

Upload PNG/JPEG through authenticated native `POST :8188/api/inputs` and use the returned image_id. `GET /api/inputs/{id}` returns dimensions, SHA 256 and reference count; `DELETE` removes unreferenced inputs. Deletion is refused while a job holds a reference.

For OpenAI-compatible `POST :8189/v1/images/generations`, pass `controlnet:canny|openpose`, `control_image_id` and `control_strength` (finite 0.01..1.5, default 1). Input must be a prepared edge/pose map matching output dimensions. Automatic OpenPose detection is not implemented. Use SDK `extra_body`. These are WAI extensions, not OpenAI standard fields.

`init_image_id` / `denoise_strength` select img2img; `reference_image_id` / `ip_adapter_strength` select IP-Adapter. Multipart `POST /v1/images/edits` is also supported. IP + manga LoRA + OpenPose + mask 1024 completed in 141.356660 seconds with 0 changed pixels outside the mask. This representative case requested 20 and executed 17 steps; it does not guarantee identity preservation or the intended edit. [IP lifetime fix and measurements](ip-lifecycle-20261019.md) · [Feature scope](feature-classification-20261019.md).

Representative console cases covered grayscale/alpha masks, grayscale soft masks, full-edit masks and masks with Canny. Soft boundaries apply to final RGB compositing; diffusion latent masks are binary. This is full-frame masked img2img with a normal WAI checkpoint, not a dedicated inpainting model or ROI-only inference. Automatic OpenPose detection from photos, multipart control_image files, arbitrary multiple LoRAs and multiple ControlNets are not provided. Existing control_image_id inputs are supported. [Mask API](mask-api.md) · [Earlier mask measurements](masked-inpainting-20261019.md).

Use `scripts/Invoke-WaiControlNet.ps1`; do not automatically retry after a timeout. Native jobs report `controlnet_evals`, `controlnet_tensors` and `gpu_controlnet_verified`. See [OpenAPI](openapi.json) for all fields.
