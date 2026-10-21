# Models

Weights are not distributed. GPL/MIT code licenses and model terms are separate.
Verify WAI's original distribution, exact version and terms before obtaining it.

| Component | Verified file | bytes |
|---|---|---:|
| WAI-Illustrious-SDXL v17.0 | waiIllustriousSDXL_v170.safetensors | 6,938,040,682 |
| Approximate VAE | taesdxl-official-f 32.safetensors |9,793,292 |

WAI SHA256:

```text
f116b0c78ff441467b0cdc8f1936e1ed18ea31e9997c7b132b1b8db533f0bd04
```

TAESDXL SHA256:

```text
ff4824aca94dd6111e0340fa749347fb74101060d9712cb5ef1ca8f1cf17502f
```

Compared the file SHA from [WAI0731's official page](https://civitai.com/models/827184?modelVersionId=2883731) and
the [version API](https://civitai.com/api/v1/model-versions/2883731).
The checkpoint includes F16 UNet, VAE and CLIP-L/G weights; inspected Safetensors header contained 2,515 tensors.
Uses this stable-diffusion.cpp version's tokenizer code/constants, without additional text-encoder files in this configuration.

Outputs use [madebyollin/taesd](https://github.com/madebyollin/taesd) TAESDXL instead of full VAE.
The verified file is F32, not merely a renamed official F16 file.
Obtaining/converting weights and checking hashes are separate from building.

The Illustrious family is associated with [Fair AI Public License 1.0-SD](https://freedevproject.org/faipl-1.0-sd/).
Also review the author's model-specific permissions and conditions.
A mirror's `cdla-permissive-2.0` label was not adopted as WAI's terms.
Publishing code does not authorize redistribution of model weights or unrestricted commercial use.

Original models reside in `/data/ai/models/`; verified native copies are under the title's `/app0/models/`.
See `config/native.json.example` for actual engine settings.
