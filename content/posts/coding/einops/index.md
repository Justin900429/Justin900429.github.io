---
title: Powerful Tools for Tensor Operations - Einops
date: 2021-07-15T15:50:37+08:00
summary: "An introduction to Einops, a powerful tool for tensor operations."
social: false
keywords:
- PyTorch
- Einops
- Tensor operations
- Deep learning
---

## Introduction

*Einops* is a powerful tool for tensor operations, including *reduce*, *rearrange*, *repeat*, and more. Its most impressive feature is its ability to make code more readable and reliable. Additionally, it includes `nn.Module` functionality, which is compatible with *PyTorch* for directly modifying tensor shapes.

In this tutorial, we'll use the MNIST dataset for demonstration purposes.

![](https://i.imgur.com/bOo6IK0.jpg)

## Installation

To install the stable version of *Einops*, use pip:

```shell
pip install einops
```

For the latest version, you can install it directly from the GitHub repository:

```shell
pip install git+https://github.com/arogozhnikov/einops
```

## Operations - Rearrange

### Changing Axes

The `rearrange` function allows you to change the shape of a tensor. For instance, treating an image as a matrix, you can perform a transpose operation. If the original shape of the tensor is `(h, w)`, transposing changes it to `(w, h)`. With Python, you can write:

```python
# Shape of data: (h, w)
transpose_data = data.transpose(0, 1)
print(transpose_data.shape)

>> torch.Size([w, h])
```

While this code is straightforward, it doesn't clearly indicate the shape transformations. Using *Einops*, the same operation is more intuitive:

```python
from einops import rearrange
data = rearrange(data, "h w -> w h")
```

Here, the transformation is described explicitly: `"h w -> w h"`. This syntax is clearer and easier to understand.

![](https://i.imgur.com/7kVIwkt.png)

### Flattening

Flattening combines multiple axes into a single dimension. For example, consider a tensor with shape `(batch_size, width, height)`:

```python
# Shape of data: (batch_size, width, height)
# Shape of stack_data: (batch_size * width, height)
stack_data = rearrange(data, "b h w -> (b h) w")
```

You can also rearrange dimensions in a non-sequential order:

```python
stack_data = rearrange(data, "b h w -> h (b w)")
```

Output examples:

<figure>
<img loading="lazy" src="https://i.imgur.com/L5emdwl.png">
<figcaption>Combining the batch and width dimensions.</figcaption>
</figure>

<figure>
<img loading="lazy" src="https://i.imgur.com/Qm54CBn.png">
<figcaption>Combining the batch and height dimensions.</figcaption>
</figure>

### Unflattening

In addition to combining axes, you can split an axis into multiple dimensions:

```python
decompose_data = rearrange(data, "(b1 b2) h w -> (b1 h) (b2 w)", b1=2)
```

![](https://i.imgur.com/jIKYQsP.png)

Here, the batch size of `4` is split into two dimensions: `b1=2` and `b2=2`.

## Operations - Reduce

The `reduce` function simplifies tensors by reducing dimensions. It shares the same syntax as `rearrange` but allows changes in dimensionality. For instance, consider a tensor with shape `(batch_size, height * h1, width * w1)`.

### Mean Averaging

Mean averaging reduces image size by averaging pixel values:

```python
reduce_img = reduce(data, "b (h h1) (w w1) -> h (b w)", "mean", h1=2, w1=2)

# Check the shape
print(f"Before: {data.shape}")
print(f"After: {reduce_img.shape}")

>>> Before: torch.Size([4, 28, 28])
>>> After: torch.Size([14, 56])
```

Output:

![](https://i.imgur.com/cnGjHPZ.png)

### Max Pooling

Similarly, you can apply max pooling to reduce size:

```python
reduce_img = reduce(data, "b (h h1) (w w1) -> h (b w)", "max", h1=2, w1=2)

# Check the shape
print(f"Before: {data.shape}")
print(f"After: {reduce_img.shape}")

>>> Before: torch.Size([4, 28, 28])
>>> After: torch.Size([14, 56])
```

![](https://i.imgur.com/gsYj7e3.png)

## Operations - Repeat

While `reduce` decreases dimensions, `repeat` increases them. For example, to expand an image tensor from `(b, h, w)` to `(b, h, w, 3)`:

```python
# Shape of data: (batch_size, width, height)
repeat_data = repeat(data, "b h w -> b h w c", c=3)

print(repeat_data.shape)

>> torch.Size([4, 28, 28, 3])
```

Duplicating dimensions can also be done easily:

```python
repeat_data = repeat(data, "b h w -> (b h) (w1 w)", w1=3)
```

![](https://i.imgur.com/JXlpJgj.png)

## PyTorch Layers

Einops operations can be integrated into PyTorch layers, making them ideal for use in models:

```python
from einops.layers.torch import Rearrange

model = nn.Sequential(
    nn.Conv2d(...),
    nn.Conv2d(...),
    ...
    Rearrange("b c h w -> b (c h w)")
    ...
)
```

## Application - Attention

Einops simplifies implementing the attention mechanism, which calculates input weights based on dot products of queries and keys. Below is an example implementation:

```python
import torch
import torch.nn as nn
from einops import rearrange

class Attention(nn.Module):
    def __init__(self, n_head: int, d_in: int, d_model: int, dropout=0.1):
        super().__init__()
        self.n_head = n_head
        assert d_model % n_head == 0
        
        self.q = nn.Linear(d_in, d_model)
        self.k = nn.Linear(d_in, d_model)
        self.v = nn.Linear(d_in, d_model)
        
        self.out = nn.Linear(d_model, d_in)
        self.dropout = nn.Dropout(p=dropout)
        self.layer_norm = nn.LayerNorm(d_model)

    def forward(self, x, mask=None):
        q = rearrange(self.q(x), 'b l (h q) -> h b l q', h=self.n_head)
        k = rearrange(self.k(x), 'b t (h q) -> h b t q', h=self.n_head)
        v = rearrange(self.v(x), 'b t (h v) -> h b t v', h=self.n_head)
        attn = torch.einsum('hblq,hbtq->hblt', [q, k]) / q.shape[-1] ** 0.5
        if mask is not None:
            attn = attn.masked_fill(mask[None], float('-inf'))
        attn = torch.softmax(attn, dim=3)
        output = torch.einsum('hblt,hbtv->hblv', [attn, v])
        output = rearrange(output, 'h b l v -> b l (h v)')
        output = self.dropout(self.out(output))
        output = self.layer_norm(output + x)
        return output
```

## References

1. [Einops Documentation](https://einops.rocks)
