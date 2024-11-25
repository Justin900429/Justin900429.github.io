---
title: 'Diffusion Transformer: A powerful archiecture for image generation'
date: 2024-09-18T18:45:59+08:00
bibFile: "content/posts/paper_reading/dit-intro/bib.json"
summary: "An introduction to the Diffusion Transformer (DiT) architecture with examples of its applications in image generation."
cover:
  image: "images/generate.png"
  alt: "DiT output"
  caption: "Generated image using DiT under 512x512 resolution."
  relative: false
social: false
keywords:
- Diffusion
- Image Generation
- Transformer
- DiT
- Diffusion Transformer
- pixart-alpha
- Generative Models
- Code
---

## Introduction

In recent developments, an increasing number of open-source models for image generation have adopted the Diffusion Transformer (DiT) {{< cite "peebles2023scalable" >}} as their backbone. Notable examples include [Stable Diffusion 3 (SD3)](https://stability.ai/news/stable-diffusion-3), [Sora](https://openai.com/index/sora/), [Open-Sora](https://github.com/hpcaitech/Open-Sora), and [FLUX](https://github.com/black-forest-labs/flux). These models demonstrate significant improvements in image quality over previous architectures. DiT represents an advancement by replacing the U-Net model traditionally used in Latent Diffusion Models (LDMs) {{< cite "rombach2022high" >}} with a transformer-based diffusion approach. In this article, we will explore what DiT is and why it has become a preferred choice for image generation.

### Why DiT?

The simple answer is: &ldquo;*it delivers better results.*&rdquo; But when we dig deeper, two key reasons stand out:

1. **Flexibility**: Transformers excel at generalizing across different modalities, making them highly versatile for generative tasks beyond image creation, such as text or audio generation. Additionally, the incorporation of the AdaLN (adaptive Layer Normalization) approach allows for the seamless integration of different input modalities, further enhancing the model's adaptability.

2. **Scalability**: DiT shows a strong correlation between model complexity and image quality. This means that as the model size increases, the generated images' quality improves significantly. DiT scales efficiently, making it a powerful tool for generating high-resolution and detailed images.

## DiT Archiecture

<figure>
<img loading="lazy" src="images/block.svg">
<figcaption>
<p>Illustration of the archiecture for DiT. Among the three archiectures, the authors adopt the one with AdaLN-zero which shows better result. Image is taken from {{< cite "peebles2023scalable" >}}</p>
</figcaption>
</figure>


### Overview

Similar to U-Net, DiT uses distinct blocks for both encoding (DiT blocks) and decoding (linear layers). Since transformers require input data to be in sequence format, the first step involves "patchifying" the image, converting it into a sequence. This sequence is then passed through several DiT blocks before being decoded by a linear layer.

### Patchifying

Patchifying is a standard operation in ViT-based architectures to transform an image into a sequence. The core idea is to collapse the height and width dimensions of the image into a single sequence dimension. However, directly converting an entire image into a sequence would result in computationally expensive operations for subsequent layers. To mitigate this, the image is first divided into smaller patches. 

Formally, given a latent representation with dimensions \\(\mathbb{R}^{C \times H \times W}\\) and a predefined patch size of \\(\mathbb{R}^{p \times p}\\), the patchified output would be \\(\mathbb{R}^{d \times (T_H \cdot T_W)}\\), where \\(T_H = H/p\\) and \\(T_W = W/p\\). Typically, \\(T_H\\) and \\(T_W\\) are equal. It's important to note that in this context, the input is referred to as "latent" because it is the output of the VAE model, rather than the original image. For instance, an image with dimensions \\(3 \times 512 \times 512\\) would be transformed to a latent of size \\(4 \times 64 \times 64\\) after passing through the VAE. Below is an example of the code used for patch embedding.

{{< collapse summary="<mark>Code for Patchifying</mark>" >}}
```python
class PatchEmbed(nn.Module):
    """Adapted from https://github.com/pprp/timm/blob/master/timm/layers/patch_embed.py"""
    def __init__(
      self,
      img_size=(64, 64),
      patch_size=(2, 2),
      in_chans=4,
      embed_dim=768,
    ):
        super().__init__()
        self.img_size = img_size
        self.patch_size = patch_size
        self.grid_size = (
          img_size[0] // patch_size[0], 
          img_size[1] // patch_size[1]
        )
        self.num_patches = self.grid_size[0] * self.grid_size[1]
        self.proj = nn.Conv2d(
          in_chans, embed_dim,
          kernel_size=patch_size, stride=patch_size
          )

    def forward(self, x):
        # Shape of x: (B, C, H, W)
        x = self.proj(x)
        # Shape of x: (B, N, D), N = (HW/p^2)
        x = x.flatten(2).transpose(1, 2)
        return x
```
{{< /collapse >}}

Although patchifying is a standard operation for vision transformers, it plays a particularly crucial role in DiT. The patch size used in DiT is often smaller than that used in typical vision transformer models. For example, state-of-the-art models use a patch size of \\(2 \times 2\\), whereas in image classification tasks, patch sizes are generally larger, such as \\(16 \times 16\\) or \\(14 \times 14\\). This difference is important because smaller patch sizes allow the model to capture finer details, which is critical for tasks like image generation where high granularity and attention to detail are essential.

### AdaLN-zero

In the figure above, we see three different types of normalization. Here, we focus on the adopted version, **AdaLN-zero**. Before diving into the architectural design, let’s first define what AdaLN-zero is. Essentially, AdaLN-zero is a variant of Adaptive Layer Norm (**AdaLN**) with zero initialization. The concept of AdaLN was first introduced by {{< cite "perez2018film" >}}. The key idea behind AdaLN is that it normalizes the input based on the conditional properties, rather than using fixed or learnable parameters shared across all inputs. More specifically, given an input \\(x \in \mathbb{R}^{N \times d}\\), a conditional input \\( c \in \mathbb{R}^{N \times d}\\) and two functions \\(f: \mathbb{R}^{N \times d} \rightarrow \mathbb{R}^{N \times d}\\) and \\(g: \mathbb{R}^{N \times d} \rightarrow \mathbb{R}^{N \times d}\\), AdaLN is defined as:
$$
AdaLN(x) = f(c) * \text{norm}(x) + g(c),
$$
where:
$$
\text{norm}(x) = \frac{x - \text{mean}(x)}{\text{std}(x) + \text{eps}}.
$$

Here, \\(*\\), \\(-\\), and \\(+\\) are element-wise operations and \\(f\\) and \\(g\\) can be nueral networks. In simpler terms, AdaLN adjusts the shape of the distribution (via \\(f\\)) and shifts its position (via \\(g\\)) according to condition, thus allowing the model to cover a broader range of input variations.

The "zero" in AdaLN-zero refers to initializing the weights for \\(f\\) and \\(g\\) to 0. This strategy, based on {{< cite "goyal2017accurate" >}}, accelerates training for large models in supervised learning settings. However, since this initialization could cause the output to always be 0, we modify the function by adding 1 to \\(f(x)\\). This ensures that the output of AdaLN-zero in the early stage is simply the normalized input:

$$
AdaLN(x) = (1 + f(c)) * \text{norm}(x) + g(c).
$$

This modification allows the model to start with the standard normalization output while gradually learning to adjust as training progresses.

{{< collapse summary="<mark>Code for AdaLN-zero</mark>" >}}
```python
class AdaLN_zero(nn.Module):
    def __init__(self, hidden_size):
        super().__init__()
        # `elementwise_affine=False` means no learnable parameters
        self.norm = nn.LayerNorm(hidden_size, elementwise_affine=False, eps=1e-6)
        self.AdaLN_modulation = nn.Sequential(
            nn.SiLU(),
            nn.Linear(hidden_size, 2 * hidden_size),
        )
        self.zero_initialization()

    def zero_initialization(self):
        nn.init.constant_(self.AdaLN_modulation[-1].weight, 0)
        nn.init.constant_(self.AdaLN_modulation[-1].bias, 0)

    def forward(self, x, c):
        # Shape of c: (B, D)
        # Shape of x: (B, N, D)
        shift, scale = self.AdaLN_modulation(c).chunk(2, dim=1)
        x = (1 + scale.unsqueeze(1)) * self.norm(x) + shift.unsqueeze(1)
        return x
```
{{< /collapse >}}

In DiT, the `scale` and `shift` are conditioned timestep and text input. This allows the normalization to adapt dynamically based on specific conditions, enhancing the model’s ability to handle diverse input scenarios.


### DiT Block

The structure of the DiT block is quite similar to that of a Vision Transformer (ViT), featuring a self-attention layer followed by a feedforward network. However, DiT introduces two key modifications:

1. An **AdaLN-zero** layer is added before both the self-attention and feedforward layers, which helps adaptively normalize the inputs.
2. Gating parameters are incorporated after the self-attention and feedforward layers to scale their outputs.

The gating parameters are also generated by a neural network, but unlike AdaLN-zero, the weights for this network are not initialized to zero, allowing for more flexibility in scaling.

{{< collapse summary="<mark>Code for DiT Block</mark>" >}}
```python
class Mlp(nn.Module):
    """ 
    Adapted from: https://github.com/huggingface/pytorch-image-models/blob/main/timm/layers/mlp.py
    """
    def __init__(
            self,
            in_features,
            hidden_features,
            act_layer=nn.GELU,
    ):
        super().__init__()
        self.fc1 = nn.Linear(in_features, hidden_features)
        self.act = act_layer()
        self.fc2 = nn.Linear(hidden_features, in_features)

    def forward(self, x):
        x = self.fc2(self.act(self.fc1(x)))
        return x


class DiTBlock(nn.Module):
    def __init__(self, hidden_size, num_heads, mlp_ratio=4.0, **block_kwargs):
        super().__init__()
        self.norm1 = nn.LayerNorm(hidden_size, elementwise_affine=False, eps=1e-6)
        self.attn = Attention(hidden_size, num_heads=num_heads, qkv_bias=True, **block_kwargs)
        self.norm2 = nn.LayerNorm(hidden_size, elementwise_affine=False, eps=1e-6)
        self.mlp = Mlp(
            in_features=hidden_size,
            hidden_features=int(hidden_size * mlp_ratio), 
            act_layer=lambda: nn.GELU(approximate="tanh"),
        )
        self.AdaLN_modulation_1 = AdaLN_zero(hidden_size)
        self.AdaLN_modulation_2 = AdaLN_zero(hidden_size)
        self.gate_scale = nn.Sequential(
          nn.SiLU(),
          nn.Linear(hidden_size, hidden_size),
        )

    def forward(self, x, c):
        # Shape of c: (B, D)
        # Shape of x: (B, N, D)
        gate_msa, gate_mlp = self.gate_scale(c).chunk(2, dim=1)
        x = x + gate_msa.unsqueeze(1) * self.attn(self.AdaLN_modulation_1(x, c))
        x = x + gate_mlp.unsqueeze(1) * self.mlp(self.AdaLN_modulation_2(x, c))
        return x
```
{{< /collapse >}}

### Decoder

In the original paper, the authors do not explicitly refer to the output layer as the "decoder." However, for easier comparison with U-Net architectures, we refer to this final layer as the decoder. The decoder consists of a linear layer followed by an **unpatchify** operation. The linear layer maps the latent representation back to the original image size, and the unpatchify operation reconstructs the image from the sequence, reversing the patchification process.

{{< collapse summary="<mark>Code for the decoder</mark>" >}}
```python
class OutputLayer(nn.Module):
    """
    The final layer of DiT.
    """
    def __init__(self, hidden_size, patch_size, out_channels):
        super().__init__()
        self.norm_final = nn.LayerNorm(
          hidden_size, 
          elementwise_affine=False, 
          eps=1e-6,
        )
        self.linear = nn.Linear(
          hidden_size, 
          patch_size * patch_size * out_channels,
        )
        self.AdaLN_modulation = AdaLN_zero(hidden_size)

    def forward(self, x, c):
        x = AdaLN_modulation(x)
        # Shape of x: (B, HW/p^2, D)
        x = self.linear(x)
        # Shape of x: (B, HW/p^2, p^2 * C)
        return x


def unpatchify(x, patch_size: int = 2, out_channels: int = 3):
    """
    x: (N, T, patch_size**2 * C)
    imgs: (N, H, W, C)
    """
    height = width = int(x.shape[1] ** 0.5)
    x = x.reshape(
        x.shape[0], 
        height,
        width, 
        patch_size, 
        patch_size,
        out_channels
      )
    x = torch.einsum('nhwpqc->nchpwq', x)
    imgs = x.reshape(
        x.shape[0], 
        out_channels,  
        height * patch_size,
        width * patch_size,
    )
    return imgs
```
{{< /collapse >}}


With this, we have a complete overview of the DiT architecture. We highly recommend users check out the full implementation [here](https://github.com/facebookresearch/DiT/blob/main/models.py).

## PixArt-\\(\alpha\\)

Training large diffusion models is computationally intensive, requiring vast resources. To reduce these costs while maintaining high performance, **PixArt-\\(\alpha\\)** {{<cite "chen2024pixartalpha">}} introduces an efficient training strategy, as illustrated in the figure below.

<figure>
<img loading="lazy" src="images/co2-dollar-lewei2.svg" width="100%">
<figcaption>
<p>Images from PixArt-\(\alpha\) {{< cite "chen2024pixartalpha" >}}. <strong>(Left)</strong>: Comparison of data usage and training time. <strong>(Right)</strong>: Comparison of <i>CO<sub>2</sub></i> emissions and training cost.</p>
</figcaption>
</figure>

PixArt-\\(\alpha\\) decomposes the training process into three stages and leverages a more efficient transformer architecture, enabling a significant boost in training efficiency without sacrificing model quality.

### Decomposing the Training Process

The authors break the training process into three distinct stages:

1. **Stage 1: Pixel Dependency Learning** – This stage focuses on learning class-conditional image generation, which is relatively straightforward and computationally inexpensive. Additionally, the model is initialized with pre-trained weights from ImageNet, further enhancing training efficiency.
   
2. **Stage 2: Text-Image Alignment** – Here, the authors refine the alignment between text descriptions and images by cleaning the data. An efficient pipeline for automatic image-text pair labeling is introduced to ensure that the text descriptions are a better match for the images.

3. **Stage 3: High-Resolution and Aesthetic Image Generation** – In this final stage, the model is trained to generate high-resolution images with improved quality. Since the model already has the capability to generate images by this stage, it converges quickly and efficiently.

### Efficient T2I Transformer

<figure>
    <img loading="lazy" src="images/pixart.svg" style="margin-left: auto; margin-right: auto;">
    <figcaption>
    <p style="text-align: center">Diffusion Transformer architecture from PixArt-\(\alpha\){{< cite "chen2024pixartalpha" >}}.</p>
    </figcaption>
</figure>

The transformer architecture in PixArt-\\(\alpha\\) incorporates several optimizations to enhance training efficiency:

1. **Cross-Attention Layer** – A cross-attention layer for text, similar to U-Net, is introduced. The **AdaLN-zero** layer handles the timestep processing, ensuring efficient temporal alignment.
   
2. **AdaLN-single** – Rather than using separate AdaLN layers for each block, all blocks share the same **AdaLN-single** layer. This simplification reduces the computational overhead while maintaining performance.

## Quick Start with Diffusers 🤗

The following example is to generate the image using the pretrained DiT model. Note that the code below is only limitted to ImageNet classes.

```python
from diffusers import DiTPipeline, DPMSolverMultistepScheduler
import torch
import numpy as np
from PIL import Image

pipe = DiTPipeline.from_pretrained("facebook/DiT-XL-2-512", torch_dtype=torch.float16)
pipe.scheduler = DPMSolverMultistepScheduler.from_config(pipe.scheduler.config)
pipe = pipe.to("cuda")

# Users should pick word from ImageNet classes
# If users are not sure, please check with `pipe.labels`
words = ["African elephant",  "gray wolf", "hotdog", "coffee mug"]

class_ids = pipe.get_label_ids(words)

output = pipe(class_labels=class_ids, num_inference_steps=50)

concat_images = [np.array(img) for img in output.images]
concat_images = np.concatenate(concat_images, axis=1)

# Save the generated image
Image.fromarray(concat_images).save("target.png") 
```

Another example is to use PixArt-\\(\alpha\\) {{< cite "chen2024pixartalpha" >}}, which also adtops DiT model. The code below is to generate images using the pretrained PixArt-\\(\alpha\\) model.

```python
from diffusers import PixArtAlphaPipeline
import torch
import numpy as np
from PIL import Image

pipe = PixArtAlphaPipeline.from_pretrained(
    "PixArt-alpha/PixArt-XL-2-512x512", 
    torch_dtype=torch.bfloat16
)
pipe = pipe.to("cuda")

# With faster speed and lower memory usage
pipe.enable_xformers_memory_efficient_attention()
prompt = [
    "A dot is jumping",
    "A cute cat", 
    "A fantasy landscape with a castle",
    "A happy family in a park",
]
output = pipe(prompt=prompt)

concat_images = [np.array(img) for img in output.images]
concat_images = np.concatenate(concat_images, axis=1)

# Save the generated image
Image.fromarray(concat_images).save("target.png") 
```

<figure>
<img loading="lazy" src="images/pixart_alpha.png">
<figcaption>
<p>Results generated with PixArt-\(\alpha\){{< cite "chen2024pixartalpha" >}}</p>
</figcaption>
</figure>

## Conclusion

DiT and PixArt-\\(\alpha\\) represent significant advancements in the field of diffusion models and image generation. DiT, with its transformer-based architecture and innovative use of AdaLN-zero, brings flexibility, scalability, and improved generation quality. PixArt-\\(\alpha\\) further enhances efficiency by breaking down the training process into stages and incorporating an optimized transformer architecture, reducing computational costs while maintaining high-resolution and aesthetic image generation. Together, these innovations pave the way for more powerful and efficient diffusion models in the future of AI-driven generative tasks.

## Reference

{{< bibliography "content/posts/paper_reading/dit-intro/bib.json" >}}