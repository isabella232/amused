# amused

![collage](./assets/collage.png)

[[Paper]]()
[[Models]]()
[[Colab]]()
[[Training Code]]()

| Model | Params |
|-------|--------|
| [amused-256](https://huggingface.co/openMUSE/diffusers-pipeline-256-finetuned) | 603M |
| [amused-512](https://huggingface.co/openMUSE/diffusers-pipeline) | 608M |

TODO - checkpoints

TODO - why/where to use amused

## 1. Usage

### Text to image

#### 256x256 model

```python
import torch
from diffusers import AmusedPipeline

pipe = AmusedPipeline.from_pretrained(
    "openMUSE/diffusers-pipeline-256-finetuned", torch_dtype=torch.float16
)  # TODO - fix path
pipe.vqvae.to(torch.float32)  # TODO - vqvae is producing nans in fp16
pipe = pipe.to("cuda")

prompt = "cowboy"
image = pipe(prompt, generator=torch.Generator('cuda').manual_seed(8)).images[0]
image.save('text2image_256.png')
```

![text2image_256](./assets/text2image_256.png)

#### 512x512 model

```python
import torch
from diffusers import AmusedPipeline

pipe = AmusedPipeline.from_pretrained(
    "openMUSE/diffusers-pipeline", torch_dtype=torch.float16
)  # TODO - fix path
pipe.vqvae.to(torch.float32)  # TODO - vqvae is producing nans n fp16
pipe = pipe.to("cuda")

prompt = "summer in the mountains"
image = pipe(prompt, generator=torch.Generator('cuda').manual_seed(2)).images[0]
image.save('text2image_512.png')
```

![text2image_512](./assets/text2image_512.png)

### Image to image

#### 256x256 model

```python
import torch
from diffusers import AmusedImg2ImgPipeline
from diffusers.utils import load_image

pipe = AmusedImg2ImgPipeline.from_pretrained(
    "openMUSE/diffusers-pipeline-256-finetuned", torch_dtype=torch.float16
)  # TODO - fix path
pipe.vqvae.to(torch.float32)  # TODO - vqvae is producing nans in fp16
pipe = pipe.to("cuda")

prompt = "apple watercolor"
input_image = (
    load_image(
        "https://raw.githubusercontent.com/huggingface/amused/main/assets/image2image_256_orig.png"
    )
    .resize((256, 256))
    .convert("RGB")
)

image = pipe(prompt, input_image, strength=0.7, generator=torch.Generator('cuda').manual_seed(3)).images[0]
image.save('image2image_256.png')
```

![image2image_256_orig](./assets/image2image_256_orig.png) ![image2image_256](./assets/image2image_256.png)

#### 512x512 model

```python
import torch
from diffusers import AmusedImg2ImgPipeline
from diffusers.utils import load_image

pipe = AmusedImg2ImgPipeline.from_pretrained(
    "openMUSE/diffusers-pipeline", torch_dtype=torch.float16
)  # TODO - fix path
pipe.vqvae.to(torch.float32)  # TODO - vqvae is producing nans in fp16
pipe = pipe.to("cuda")

prompt = "winter mountains"
input_image = (
    load_image(
        "https://raw.githubusercontent.com/huggingface/amused/main/assets/image2image_512_orig.png"
    )
    .resize((512, 512))
    .convert("RGB")
)

image = pipe(prompt, input_image, generator=torch.Generator('cuda').manual_seed(15)).images[0]
image.save('image2image_512.png')
```

![image2image_512_orig](./assets/image2image_512_orig.png) ![image2image_512](./assets/image2image_512.png)

### Inpainting

#### 256x256 model

```python
import torch
from diffusers import AmusedInpaintPipeline
from diffusers.utils import load_image
from PIL import Image

pipe = AmusedInpaintPipeline.from_pretrained(
    "openMUSE/diffusers-pipeline-256-finetuned", torch_dtype=torch.float16
)  # TODO - fix path
pipe.vqvae.to(torch.float32)  # TODO - vqvae is producing nans with this example when in fp16
pipe = pipe.to("cuda")

prompt = "a man with glasses"
input_image = (
    load_image(
        "https://raw.githubusercontent.com/huggingface/amused/main/assets/inpainting_256_orig.png"
    )
    .resize((256, 256))
    .convert("RGB")
)
mask = (
    load_image(
        "https://raw.githubusercontent.com/huggingface/amused/main/assets/inpainting_256_mask.png"
    )
    .resize((256, 256))
    .convert("L")
)    

for seed in range(20):
    image = pipe(prompt, input_image, mask, generator=torch.Generator('cuda').manual_seed(seed)).images[0]
    image.save(f'inpainting_256_{seed}.png')

```

![inpainting_256_orig](./assets/inpainting_256_orig.png) ![inpainting_256_mask](./assets/inpainting_256_mask.png) ![inpainting_256](./assets/inpainting_256.png)

#### 512x512 model

```python
import torch
from diffusers import AmusedInpaintPipeline
from diffusers.utils import load_image

pipe = AmusedInpaintPipeline.from_pretrained(
    "openMUSE/diffusers-pipeline", torch_dtype=torch.float16
)  # TODO - fix path
pipe.vqvae.to(torch.float32)  # TODO - vqvae is producing nans with this example when in fp16
pipe = pipe.to("cuda")

prompt = "fall mountains"
input_image = (
    load_image(
        "https://raw.githubusercontent.com/huggingface/amused/main/assets/inpainting_512_orig.jpeg"
    )
    .resize((512, 512))
    .convert("RGB")
)
mask = (
    load_image(
        "https://raw.githubusercontent.com/huggingface/amused/main/assets/inpainting_512_mask.png"
    )
    .resize((512, 512))
    .convert("L")
)
image = pipe(prompt, input_image, mask, generator=torch.Generator('cuda').manual_seed(0)).images[0]
image.save('inpainting_512.png')
```

![inpainting_512_orig](./assets/inpainting_512_orig.jpeg) 
![inpainting_512_mask](./assets/inpainting_512_mask.png) 
![inpainting_512](./assets/inpainting_512.png)

## 2. Performance

Amused inherits performance benefits from original [muse](https://arxiv.org/pdf/2301.00704.pdf). 

1. Parallel decoding: The model follows a denoising schedule that aims to unmask some percent of tokens at each denoising step. At each step, all masked tokens are predicted, and some number of tokens that the network is most confident about are unmasked. Because multiple tokens are predicted at once, we can generate a full 256x256 or 512x512 image in around 12 steps. In comparison, an autoregressive model must predict a single token at a time. Note that a 256x256 image with the 16x downsampled VAE that muse uses will have 256 tokens.

2. Fewer sampling steps: Compared to many diffusion models, muse requires fewer samples.

Additionally, amused uses the smaller CLIP as its text encoder instead of T5 compared to muse. Amused is also smaller with ~600M params compared the largest 3B param muse model.

![a100_bs_1](./assets/a100_bs_1.png)
![a100_bs_8](./assets/a100_bs_8.png)
![4090_bs_1](./assets/4090_bs_1.png)
![4090_bs_8](./assets/4090_bs_8.png)

### torch.compile

### flash attention

### fused rms norm blocks

## 3. Training

## 4. Finetuning

## 5. Acknowledgements

## 6. Citation
