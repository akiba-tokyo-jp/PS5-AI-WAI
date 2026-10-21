# Builds and reproducibility

Rebuilt the latest normal/debug, bounded-storage version from public source and verified SELF integrity. See [logging verification](logging.md) for the distinction between console results and the separate public build.

## Verified scope

Recompiled 13 native translation units and 1 RADV platform translation unit from the public tree,
linked the existing SHA 256-pinned SD/GGML/RADV/SDK dependencies, and verified ELF→native ELF→SELF conversion and inspection.
Windows API tests use fixtures without inference. They are separate from evidence of PS5 image generation.

**A clean build of every dependency and automated first installation are not complete.**
This repository publishes sources, matching modifications and build profiles for the successful console version.
Cloning into an environment without the dependency cache does not produce an executable in 1 command.
Models, SDKs, generated shaders, libraries and libc.prx are not committed to Git.

## Tools and sources

Verified host: Windows x64, Python 3.13, llvm-mingw 20261006 and MSYS 2 CLANG 64 LLVM23.
The native SDK combines public SDK v0.42 with PS5_PayloadSDK `b83202be…` headers/platform;
the CPU gateway uses separate public SDK v0.44. Do not mix these 2 CRT/header sets.
Used Vulkan-Headers `c46850864f4661461b0f6cb9922c058ffea4915e`.

Dependency URLs, revisions and SDK archive SHA values are in the [lock file](../dependencies.lock.json).
Use a work drive with sufficient free space.

```powershell
python tools/prepare_sources.py --out D:/ps5-dev/sources
```

Existing modified checkouts are not overwritten. For attempt 2, use another output directory or a checkout with its changes preserved.

## Dependency build origins

| Input | Origin and modifications |
|---|---|
| PS5 SDK / libps 5platform | Public v0.42 SDK plus pinned fork include/freebsd, include/khronos, include/ps 5 and platform headers; platform/src, regex and dlmalloc/mspace compiled into PS5 static libraries |
| RADV | Pinned PS5_Mesa, ACO and PS5 winsys; LLVM target disabled. Build Mesa host mesa_clc/vtn_bindgen 2 on Windows separately from target libraries |
| RADV modifications | Physical-memory and imported-heap bookkeeping in `patches/mesa/overlay`; final winsys platform replaced by `native/app/radv_ps5_platform_native.c` |
| stable-diffusion / GGML | Pinned SD.cpp with Vulkan; dynamic backends, OpenMP and native CPU variants disabled; apply `patches/stable-diffusion/overlay` |
| SD/GGML modifications | Direct host import, graph cuts, disk weight retirement, loader polling, scratch/graph admission; match internal core-runner/allocator ABI |
| Native CRT / SELF tools | Pinned PS5_Vulkan `tooling/native`; host packager built with C++20 and zlib |
| Compatibility functions / AGC stubs | `native/link/compat.c` and public stub sources; link stubs are not GPU implementations |
| libc.prx | PS5_Vulkan public clean-room native-libc procedure; match the verified SDK/runtime configuration |

Target: `x86_64-sie-ps5`, FreeBSD/x86-64,16KiB pages, emulated TLS and IEEE denormal settings.
Mesa settings: `radv-winsys=ps5`, `vulkan-drivers=amd`, `llvm=disabled`, `amd-use-llvm=false`, `buildtype=debugoptimized`, `b_ndebug=true`.
Avoid Windows text conversion in host shader tools and validate generated SPIR-V.
This source release does not yet include a general script integrating all dependency generation.

SD enables AVX/AVX 2/FMA/F16C/BMI 2, disables AVX 512, and uses C++20, public libc++ and exceptions.
Changing `GGML_MAX_NAME=160` or `sizeof(ggml_tensor)=432` breaks the native ABI match.
Use the [configuration](../config/native.json.example) for approximate VAE/Flash Attention, batched CFG and model-residency settings.

## Native rebuild with reviewed dependencies

1. Copy `config/build.json.example` to a private file such as `.local/build.json` and configure paths.
2. Populate `DEPS` with libraries matching names/SHA values in [build-profiles/native.json](../build-profiles/native.json).
3. Specify the native SDK, LLVM, Vulkan-Headers, pinned upstream source and host packager.
4. Run with a fresh output directory.

```powershell
python tools/build.py --config .local/build.json --precision fp32 --out D:/ps5-dev/build/wai-native
```

Historical personal logs, PIDs and session receipts are unnecessary. Recompile sources, replace the driver's platform member and link.
The default is the measured faster FP32 version. Build the FP16 experiment with `--precision fp16`.
For FP16, unset `GGML_VK_DISABLE_F16` and inspect actual startup logs and physical features.
Use direct convolution only after GPU readback matches independent CPU reference values.
Recompile the core runner from the public overlay and replace the corresponding 1 member of the pinned SD dependency archive.
Verify identical PT_LOAD content before/after stripping and 6 native runtime providers.
Completed outputs are not overwritten. Dependency SHA mismatches stop the build.
Do not make libraries built with another toolchain pass by merely editing expected hashes.
Such a change requires regenerating dependencies and revalidating ABI, constructors and imports.

Outputs are `title/PPSA99188/eboot.bin`, icon and param metadata.
The command does not automatically assemble or deploy the complete installed title with runtime libc, licenses, configuration and models.
File existence does not prove PS5 launch or GPU execution. Check [deployment prerequisites](deployment.md).

## OpenAI adapter and host tests

```powershell
python tools/build_gateway.py --config .local/build.json --out D:/ps5-dev/build/gateway
# Add PAYLOAD_SDK and SDK_SOURCE to the configuration to also build the PS5 ELF
python tools/build_gateway.py --config .local/build.json --out D:/ps5-dev/build/gateway --native
```

The gateway payload CRT retains the SDK's own syscall-range setup while omitting credential-changing helpers.
Startup is not a complete no-op. Do not replace the GPU native application's CRT with it.
Builds do not connect to the PS5.

```powershell
$env:WAI_TEST_OUT = 'D:/ps5-dev/build/gateway'
$env:WAI_TEST_LLVM = 'D:/ps5-dev/llvm-mingw/bin'
python tests/openai_gateway_protocol.py
python tests/native_log_storage.py
python tests/native_precision.py --config .local/build.json --out D:/ps5-dev/build/precision-host
```

Tests cover HTTP authentication, unsupported parameters, concurrent requests, backend failure, HMAC/base 64, log rotation/storage exhaustion and retained-state reporting.
Host fixtures return no generated image and do not establish successful GPU inference.

## Host logging-policy checks

The following are CPU mock checks without model inference or GPU calls. Select an output location on a drive such as D.

```powershell
$env:WAI_TEST_LLVM = 'D:/ps5-dev/llvm/bin'
$env:WAI_TEST_SD = 'D:/ps5-dev/sources/stable-diffusion'
$env:WAI_TEST_LOG_OUT = 'D:/ps5-dev/tests/log-policy'
$env:WAI_TEST_STDIO_OUT = 'D:/ps5-dev/tests/log-stdio'
python tests/native_log_policy.py
python tests/native_log_stdio.py
```

Using the actual policy parser and driver logger, verify omitted formatting in normal mode and batched synchronization in debug mode,
critical/checkpoint synchronization and sticky fsync failures.
The API header tests cover repeated rotation and preserving bounds when rename fails.

## Additional manga LoRA model

The initial manga LoRA version used 7GiB Direct /6.75GiB Vulkan budget. The current profile uses 9422MiB (about 9.201GiB) Direct /8.75GiB Vulkan budget and a WAI v17-specific converted LoRA. Obtain the base WAI and author imge's GLoRA separately, preserving terms and attribution. Weights are not bundled on GitHub.

```powershell
python -m pip install numpy
python tools/prepare_manga_lora_v1.py --base D:/PS5-AI/models/waiIllustriousSDXL_v170.safetensors --directory D:/PS5-AI/models/loras
```

This is Windows model-format conversion, not image generation. Verify the pinned mirror commit and original-distribution SHA; record the full WAI SHA in conversion metadata. Convert the original GLoRA correction to rank 48/F32 and sample-check corrections across 1008 used layers. No SVD or rank truncation is applied. Conversion is checkpoint-specific and assumes clip_skip=2.

Place `manga-illustrious-wai-v17-r48-f32.safetensors` in the title's models directory and use updated `config/native.json.example`. Updating only the old 6GiB config is insufficient: rebuild native code and the in-app RADV VA split together. Deploy after normal OS close, original backups and hash verification.


## ControlNet rebuild and protocol tests

The profile recompiles 7 core members (runner/image/diffusion_engine/model_manager/model_builders/photomaker_extension/ggml_graph_cut). They share conditioning, CLIP and ControlNet class definitions; do not mix old core members. The authoritative model_manager is upstream `src/model_manager.cpp`. Direct 9422MiB (about 9.201GiB),19 regions comprising 18×512MiB +206MiB, and an initial 1GiB CPU reservation. CPU CLIP weight groups target 16MiB without splitting large individual tensors. Supplied character image files are included.

The optimization candidate includes releasing scratch ownership before CPU CLIP. The build copies the pinned GGML Vulkan archive and replaces only `ggml-vulkan.cpp.o`. Other members and generated SPIR-V remain unchanged; checks verify changed members and matching engine/model_manager/GGML C API definitions. Set `GGML_SHADER_DIR` to the directory containing pinned `ggml-vulkan-shaders.hpp`; set `SPIRV_HEADERS` to the public SPIRV-Headers root containing `include/spirv/unified1/spirv.hpp`. SHA mismatches stop the build.

ControlNet batch-pair is `1` in the profile, reducing 40 ControlNet evaluations to 20 over 20 steps. Adopted native 256/1024 generation, finite values and reference PNG identity are recorded in [this verification](optimization-20261019.md). The public source rebuild also passed; this does not mean that separate public SELF was executed on PS5.

```powershell
python tests/optimization_build_profile.py --config .local/build.json --out D:/ps5-build/optimization-profile-check --expected-controlnet-batch-pair 1
```

This checks configuration, source hashes, API definitions and archive replacement. It does not replace native compilation or PS5 execution.

```powershell
python tools/build_controlnet_tests.py --config .local/build.json --out D:/ps5-build/protocol-fixtures
$env:PS5_WAI_FIXTURE_DIR='D:/ps5-build/protocol-fixtures'
$env:PS5_WAI_TEST_OUT='D:/ps5-build/protocol-results'
python tests/controlnet_protocol.py
$env:PS5_WAI_TEST_OUT='D:/ps5-build/client-results'
python tests/controlnet_powershell_client.py
```

Requires Python/Pillow and Windows PowerShell or pwsh; optionally set PS5_WAI_POWERSHELL to the executable. Tests use only localhost 18188/18189/18289. The fixture engine deliberately returns failure, without inference or PS5 connections. See [ControlNet measurements](controlnet.md) for actual PS5 evidence.
