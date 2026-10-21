# Feature verification and improvement classification (2026-10-19)

Verification against saved console logs, PNGs and current source. V18 preserves Direct 9422MiB (about 9.201GiB) and an 8.75GiB inference budget, fixing IP weight lifetimes and CPU encoder order. Representative normal features passed up to 1024; this is not exhaustive parameter/combination coverage. Verified operator recovery after disconnect and 6 consecutive representative jobs, not 24-hour or full-catalog completion.

A256 success verifies mechanisms such as API, GPU work and mask preservation, not 1024 or visual quality. Even at 1024, completion is assessed separately from identity, pose and requested colors. Windows only preprocesses/transfers inputs and retrieves/compares PNGs; inference runs on PS5. No verification videos are recorded.

## Verified scope

| Feature / combination | Saved console result | Quality / unverified scope | Classification |
| --- | --- | --- | --- |
| WAI text-to-image + manga LoRA + OpenPose | V18:6 consecutive 1024/20-step jobs, 124.176–124.446 seconds, median 124.199. Same-prompt PNG pairs matched | Representative 258–260B short prompts; not the original long-prompt catalog conditions or 24-hour operation | Mechanisms/operation verified for 6 representative jobs |
| img2img + manga LoRA + OpenPose | Generated 3 poses at 1024 from supplied character in 108–115 seconds | Hair/outfit broadly reflected; face, sleeves and body varied. Requested hands-on-knees sitting pose was not achieved | Quality needs improvement |
| IP-Adapter alone, no LoRA, strength 0.3 | V17:256 in 38.686 seconds, 1024 in 95.859. V18 first 1024 in 126.311:20steps, 16 image tokens, GPU IP attention and PNG retrieval; identical to the matched V17 1024 PNG produced by V18 | Different first-run/warm states prevent a slowdown-rate claim. Exact identity and both-arms-down instruction were not achieved | V18 regression passed; quality needs improvement |
| IP-Adapter + OpenPose, no LoRA, IP strength 0.7 | V18:1024 in 146.823 seconds, 20steps, 40 ControlNet evaluations, 16 image tokens, both GPU paths and PNG retrieval | Both arms lowered, but face was pale/high-contrast; style, long sleeves and uncropped shoes were not achieved. Prompt and other conditions differ from IP 0.3 alone |1024 mechanism verified; quality target unmet |
| IP-Adapter + masked img2img, no LoRA |256 in 31.056 seconds: 17 actual steps, mask preparation, GPU IP attention and in-mask changes. IP+mask 1024 verified in the all-features case below | No separate 1024 test without LoRA/ControlNet. Identity lock and practical edit quality remain unmet | Representative mechanism verified; quality needs improvement |
| IP-Adapter + manga LoRA + OpenPose + mask | V17:256 in 51.456 seconds; V17 1024 allocation failed. V18:1024 completed in 141.357 within the same reservation: 17 actual steps, 34 ControlNet evaluations, 16 image tokens, both GPU paths and 0 outside-mask changes | Preserved outside-mask face/arms prove masking, not identity lock or pose editing. Prompt requested blue clothing, not a red-edit test |1024 allocation fixed and retested; editing quality needs improvement |
| Grayscale partial mask + manga LoRA |256 in 63.217 seconds; 0 outside-mask changes. No-ControlNet 1024 covered by the soft grayscale mask below | Requested red skirt was not achieved | Mechanism verified; semantic editing needs improvement |
| Alpha partial mask + manga LoRA + OpenPose |1024 in 172.897 seconds; 0 outside-mask changes and GPU ControlNet verified | Requested color was not achieved | Mechanism verified; quality needs improvement |
| Soft grayscale/alpha mask + manga LoRA |256:37.572 /14.410 seconds, identical PNG from equivalent logical masks. V18 grayscale 1024:98.782 seconds, 17 actual steps; 0 changes among 902,575 outside pixels; 120,373 changed inside pixels; 0 final-RGB blending violations across 77,273 intermediate-alpha pixels | Red skirt remained unmet even without ControlNet. Diffusion uses binary masks; softness applies at final RGB. Alpha 1024 not separately generated here; supplemented by 256 equivalence and prior alpha 1024 |1024 mechanism/final feather verified; semantic editing/latent soft masks need improvement |
| Full-edit mask + manga LoRA |256 in 14.435 seconds. V18:1024 in 75.634,17 actual steps, mask_prepared=true, 1,036,664 changed pixels. All-white mask correctly needed 0 latent blends. GPU generation/PNG retrieval passed |1024 faces/hands improved over 256, but red skirt remained unmet. All-white has no preserved area and cannot prove outside-mask preservation |1024 mechanism verified; semantic editing needs improvement |
| Partial mask + manga LoRA + Canny | V17:256 in 51.356 seconds, 6,526 changed inside pixels. V18:1024 in 151.549,17 actual steps, 34 ControlNet evaluations; 0 changes among 944,203 outside pixels; 104,351 changed inside pixels. GPU ControlNet/PNG identity passed | Red-skirt instruction unmet |1024 mechanism verified; semantic editing needs improvement |
| Canny / OpenPose | Prior GPU generation at 256/1024 for each; V18 also verified IP+OpenPose 1024. Inputs are prepared RGB control maps | Not exhaustive arbitrary feature combinations/parameters | Individual and representative combined cases verified |
| OpenAI connection interrupted | Closed Windows HTTP at sampler progress 0.32; PS5 completed in 142.043 seconds. Operator identified ID in internal logs and recovered identical gateway/native PNGs over a separate GET. Next normal generation passed in the same process | Unknown-job discovery before response is neither verified nor provided to normal clients. Not a cancellation/GPU-stop test | Continued execution/operator recovery verified; correlation IDs remain open |
| Normal OS app close/restart and stale records | V18 update preserved boot/gateway, normally closed and launched native 1 time. Converted 1 each of synthetic queued/running records to failed; 2 real records/PNGs remained identical. Deleted only synthetic records; startup 64MiB Compute had 0 mismatches | Did not forcibly interrupt real inference; distinct from GPU-hang or sudden-power-loss recovery | Normal restart and synthetic stale-record handling verified |

Times are internal PS5 values; ratios across different load states, resolutions or features are not speedups. V18's3 IP cases and grayscale-soft/full/Canny-mask 1024 completed successfully with 0 nonfinite values, final CPU CLIP borrowing live=0 and fault_retained=false. Each condition was observed 1 time, not long-term operation. Img2img denoise_strength=0.8 executes 17 of 20 requested steps. Representative normal/edge cases were selected, not every combination of mechanism parameters.

## Operational verification and limitations

Disconnected only Windows HTTP61.938 seconds after POST at sampler progress 0.32. Comparing the 20-step callback and completed GPU UNet source establishes at least 6 completed sampler steps. PS5 completed in 142.042663 seconds. Operator privileges and existing gateway correspondence logs supplied the ID, not the client's HTTP response. A separate GET recovered matching gateway/native PNGs. This does not provide reliable unknown-job discovery to normal clients before a response. Correlation IDs, idempotency and advance job IDs are candidates. Unknown-outcome generation POSTs are not automatically resent.

Then, in the same native/gateway processes, generated 6 serial 1024×1024/20steps/CFG 5/Euler A/manga LoRA 1+OpenPose 1 jobs: 124.176034,124.446369,124.189468,124.195161,124.315023,124.203206 PS5 seconds. First/last 3 medians 124.189468/124.203206; ratio 1.000111. Used 3 short ribbon prompts in A/B/C/A/B/C order; same-prompt PNGs matched byte for byte. All passed finite checks, final CPU Direct borrowing 0 and fault_retained=false; final owned Direct usage was 8,170,340,352B in every job.

The original slowdown used 768×1024 and approximately 1.7–2.0KB catalog prompts; these 3 prompts were 1024×1024 and 258–260 UTF-8 bytes. The 4–5× slowdown did not recur across the 4th–6th-job range observed here, but this is not an exact reproduction, 24-hour test or full 2,756-image completion. Phase timings are CPU wall time, not pure GPU timestamps.

After 6 jobs, current logs were engine 5,533,681B, startup 4,100,804B and pool 1,231,115B, within their 16/4/4MiB bounds. This did not force-test every log series or a full disk. Long-term image/history storage requires separate management.

## Memory-lifetime fix for all-features 1024

Saved measurements distinguish fragmented free regions from insufficient total Vision capacity. V18 makes only IP projection and UNet IP K/V reloadable and runs CLIP-Vision after freeing CPU text weights. Reservation, GPU driver, precision, shaders and API remain unchanged.

Manga LoRA 1+IP 0.3+OpenPose 1+partial-mask 1024 completed in 141.356660 seconds. OpenPose was already loaded; text conditioning was actually recomputed. All 222 text-weight and 5 Vision-weight allocations were returned; logs prove their lifetimes no longer overlap. Direct peak 9,313,894,400 bytes left 565,788,672 within the reservation. This pool highwater includes CPU borrowing, not dedicated VRAM.

See [IP lifetime verification](ip-lifecycle-20261019.md) and [machine-readable data](ip-lifecycle-20261019-evidence.json) for causal values, first/warm comparison conditions, PNG identity and public-build/executed-SELF distinctions. Raw failure logs and private reference images are not attached.

## Not provided by the current implementation

| Item | Current handling | Future possibility |
| --- | --- | --- |
| Strict identity lock and guaranteed requested hands/poses | No guaranteed-result feature | Consider trained character LoRA, reference methods, appropriate skeleton maps/strengths. These few images do not establish impossibility |
| Automatic OpenPose from photos | Not implemented; accepts prepared skeleton maps | Additional preprocessor implementation |
| Server-side Canny preprocessing | Accepts prepared RGB edge maps; verification preprocessing used a Windows image filter | PS5 preprocessing requires development |
| Multipart control_image files | Not implemented; existing control_image_id works | Add reception, ownership and cleanup |
| Continuous soft masks during diffusion | Current masks are rounded to binary; final RGB uses 0–255 feathering | Requires latent changes/revalidation |
| Dedicated inpainting weights / ROI-only inference | Full-frame masked img2img with normal WAI checkpoint | Different model compatibility, quality and memory validation |
| Arbitrary multiple LoRAs / concurrent multiple ControlNets | API selects configured manga LoRA and 1 ControlNet type | Extend API, residency and budgets |
| All OpenAI products, streaming, n>1, transparent/JPEG/WebP output | Outside this API; compatible subset returns 1 PNG | Separate feature implementations |
| Cancel active jobs / automatic GPU-reset recovery | No cancellation API; idle shutdown only. GPU faults retain resources, degrade and do not retry | Never force-free uncertain active resources; safe cancellation requires development |
| Unknown-job discovery/idempotency after pre-response OpenAI disconnect | Not guaranteed by client API; operator recovered ID from internal logs | Client correlation IDs, idempotency keys or advance acceptance IDs |

Not implemented, failed in the tested configuration, and fundamentally impossible are different claims. Current evidence does not establish impossibility of these quality targets/combinations. No inferred temperature, clock or utilization display is added.

## Classification and remaining improvements/tests

- Fixed and retested: Vision allocation failure in all-features 1024. Normal generation, references, masks and ControlNet passed representative console conditions above.
- Functional but quality needs improvement: requested masked colors, face/hand edits, identity and skeleton following. Successful PNGs or exact outside-mask preservation do not establish intended-edit success.
- Not currently provided: preprocessors, multiple ControlNets, dedicated inpainting/ROI, continuous latent soft masks, cancellation and correlation IDs above. These need development/testing and are not judged fundamentally impossible.
- Further console tests: long continuous operation such as 24 hours, exact original long-prompt catalog conditions, full disks, GPU faults and sudden-power-loss recovery. Here real inference was not forcibly interrupted; normal OS restart with synthetic records and client disconnect were tested. The 36 host reload checks are not GPU-hang recovery evidence.

## Published verification scope

Conditions/limitations are in [IP memory lifetime and combined 1024](ip-lifecycle-20261019.md), [machine-readable success values](ip-lifecycle-20261019-evidence.json), [earlier mask tests](masked-inpainting-20261019.md) and [performance comparison](optimization-20261019.md). This update includes no new private reference/production images, personal logs or credentials. No verification video was created.

## Operational measurements

[Console operations/recovery](operations-recovery-20261019.txt) · [Machine-readable data](operations-recovery-20261019-evidence.json). All 6 jobs ended at 8,170,340,352 Direct bytes. Process-lifetime peak 9,705,046,016 left approximately 166.55MiB within the reservation when observed. The highwater includes CPU borrowing, not dedicated VRAM.6 stable jobs do not guarantee 24 hours or arbitrary combinations.
