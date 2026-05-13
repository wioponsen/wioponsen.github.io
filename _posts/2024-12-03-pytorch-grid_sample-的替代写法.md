---
title: 'pytorch grid_sample 的替代写法'
date: 2024-12-03 09:16:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

ref:
1. https://zenn.dev/pinto0309/scraps/7d4032067d0160
2. https://github.com/ibaiGorordo/CREStereo-Pytorch/blob/b6c7a9fe8dc2e9e56ba7b96f4677312309282d15/nets/utils/utils.py#L35

因为 GridSample op 在不同平台支持或者opset支持是不一样的， 有些设备上无法支持， 可以使用该方法，用别的op替换 GridSample

```python
# Ref: https://zenn.dev/pinto0309/scraps/7d4032067d0160
def bilinear_grid_sample(im, grid, align_corners=False):
    """Given an input and a flow-field grid, computes the output using input
    values and pixel locations from grid. Supported only bilinear interpolation
    method to sample the input pixels.

    Args:
        im (torch.Tensor): Input feature map, shape (N, C, H, W)
        grid (torch.Tensor): Point coordinates, shape (N, Hg, Wg, 2)
        align_corners {bool}: If set to True, the extrema (-1 and 1) are
            considered as referring to the center points of the input’s
            corner pixels. If set to False, they are instead considered as
            referring to the corner points of the input’s corner pixels,
            making the sampling more resolution agnostic.

    Returns:
        torch.Tensor: A tensor with sampled points, shape (N, C, Hg, Wg)
    """
    n, c, h, w = im.shape
    gn, gh, gw, _ = grid.shape
    assert n == gn

    x = grid[:, :, :, 0]
    y = grid[:, :, :, 1]

    if align_corners:
        x = ((x + 1) / 2) * (w - 1)
        y = ((y + 1) / 2) * (h - 1)
    else:
        x = ((x + 1) * w - 1) / 2
        y = ((y + 1) * h - 1) / 2

    x = x.view(n, -1)
    y = y.view(n, -1)

    x0 = torch.floor(x).long()
    y0 = torch.floor(y).long()
    x1 = x0 + 1
    y1 = y0 + 1

    wa = ((x1 - x) * (y1 - y)).unsqueeze(1)
    wb = ((x1 - x) * (y - y0)).unsqueeze(1)
    wc = ((x - x0) * (y1 - y)).unsqueeze(1)
    wd = ((x - x0) * (y - y0)).unsqueeze(1)

    # Apply default for grid_sample function zero padding
    im_padded = torch.nn.functional.pad(im, pad=[1, 1, 1, 1], mode='constant', value=0)
    padded_h = h + 2
    padded_w = w + 2
    # save points positions after padding
    x0, x1, y0, y1 = x0 + 1, x1 + 1, y0 + 1, y1 + 1

    # Clip coordinates to padded image size
    x0 = torch.where(x0 < 0, torch.tensor(0, device=im.device), x0)
    x0 = torch.where(x0 > padded_w - 1, torch.tensor(padded_w - 1, device=im.device), x0)
    x1 = torch.where(x1 < 0, torch.tensor(0, device=im.device), x1)
    x1 = torch.where(x1 > padded_w - 1, torch.tensor(padded_w - 1, device=im.device), x1)
    y0 = torch.where(y0 < 0, torch.tensor(0, device=im.device), y0)
    y0 = torch.where(y0 > padded_h - 1, torch.tensor(padded_h - 1, device=im.device), y0)
    y1 = torch.where(y1 < 0, torch.tensor(0, device=im.device), y1)
    y1 = torch.where(y1 > padded_h - 1, torch.tensor(padded_h - 1, device=im.device), y1)

    im_padded = im_padded.view(n, c, -1)

    x0_y0 = (x0 + y0 * padded_w).unsqueeze(1).expand(-1, c, -1)
    x0_y1 = (x0 + y1 * padded_w).unsqueeze(1).expand(-1, c, -1)
    x1_y0 = (x1 + y0 * padded_w).unsqueeze(1).expand(-1, c, -1)
    x1_y1 = (x1 + y1 * padded_w).unsqueeze(1).expand(-1, c, -1)

    Ia = torch.gather(im_padded, 2, x0_y0)
    Ib = torch.gather(im_padded, 2, x0_y1)
    Ic = torch.gather(im_padded, 2, x1_y0)
    Id = torch.gather(im_padded, 2, x1_y1)

    return (Ia * wa + Ib * wb + Ic * wc + Id * wd).reshape(n, c, gh, gw)

```

替换op为：
```shell
┏━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┓
┃                 ┃ Original Model ┃ Simplified Model ┃
┡━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━┩
│ Add             │ 15             │ 13               │
│ Cast            │ 11             │ 6                │
│ Concat          │ 1              │ 0                │
│ Constant        │ 52             │ 13               │
│ ConstantOfShape │ 5              │ 0                │
│ Div             │ 2              │ 2                │
│ Equal           │ 4              │ 0                │
│ Expand          │ 4              │ 4                │
│ Floor           │ 2              │ 2                │
│ Gather          │ 2              │ 2                │
│ GatherElements  │ 4              │ 4                │
│ Greater         │ 4              │ 4                │
│ Identity        │ 7              │ 0                │
│ Less            │ 4              │ 4                │
│ Mul             │ 18             │ 12               │
│ Pad             │ 1              │ 1                │
│ Reshape         │ 6              │ 4                │
│ Slice           │ 1              │ 0                │
│ Sub             │ 8              │ 4                │
│ Transpose       │ 1              │ 0                │
│ Unsqueeze       │ 8              │ 8                │
│ Where           │ 12             │ 8                │
│ Model Size      │ 11.8KiB        │ 8.9KiB           │
└─────────────────┴────────────────┴──────────────────┘


```
替换op为：
Add、Cast、Constant、Div、Expand、Floor、Gather、GatherElements、Greater、Less、Mul、Pad、Reshape、Unsqueeze、Where

其中，使用clip可以简化 Greater 和Less,通过reshape,简化 Unsqueeze
通过Gather、take等可以简化掉GatherElements
```python
def bilinear_grid_sample(im, grid, align_corners=False):

    n, c, h, w = im.shape
    gn, gh, gw, _ = grid.shape
    assert n == gn

    x = grid[:, :, :, 0]
    y = grid[:, :, :, 1]

    if align_corners:
        x = ((x + 1) / 2) * (w - 1)
        y = ((y + 1) / 2) * (h - 1)
    else:
        x = ((x + 1) * w - 1) / 2
        y = ((y + 1) * h - 1) / 2

    x = x.view(n, 1, -1)  # [N, 1, hw]
    y = y.view(n, 1, -1)

    x0 = torch.floor(x).long()
    y0 = torch.floor(y).long()
    x1 = x0 + 1
    y1 = y0 + 1

    wa = ((x1 - x) * (y1 - y))  # [N, 1, hw]
    wb = ((x1 - x) * (y - y0))
    wc = ((x - x0) * (y1 - y))
    wd = ((x - x0) * (y - y0))

    # Apply default for grid_sample function zero padding
    im_padded = torch.nn.functional.pad(im, pad=[1, 1, 1, 1], mode='constant', value=0)
    padded_h = h + 2
    padded_w = w + 2
    # save points positions after padding
    x0, x1, y0, y1 = x0 + 1, x1 + 1, y0 + 1, y1 + 1

    # Clip coordinates to padded image size
    x0 = torch.clip(x0, 0, padded_w - 1)
    x1 = torch.clip(x1, 0, padded_w - 1)
    y0 = torch.clip(y0, 0, padded_h - 1)
    y1 = torch.clip(y1, 0, padded_h - 1)

    im_padded = im_padded.view(n, c, -1)  # [N, C, HW]

    # # 1
    # x0_y0 = (x0 + y0 * padded_w).expand(-1, c, -1)  # [N, C, hw]
    # x0_y1 = (x0 + y1 * padded_w).expand(-1, c, -1)
    # x1_y0 = (x1 + y0 * padded_w).expand(-1, c, -1)
    # x1_y1 = (x1 + y1 * padded_w).expand(-1, c, -1)

    # Ia = torch.gather(im_padded, 2, x0_y0)  # [N, C, hw]
    # Ib = torch.gather(im_padded, 2, x0_y1)
    # Ic = torch.gather(im_padded, 2, x1_y0)
    # Id = torch.gather(im_padded, 2, x1_y1)

    # # 2
    # x0_y0 = (x0 + y0 * padded_w)  # [N, 1, hw]
    # x0_y1 = (x0 + y1 * padded_w)
    # x1_y0 = (x1 + y0 * padded_w)
    # x1_y1 = (x1 + y1 * padded_w)

    # Ia = torch.zeros([n, c, gh*gw]).type_as(im_padded)
    # Ib = torch.zeros([n, c, gh*gw]).type_as(im_padded)
    # Ic = torch.zeros([n, c, gh*gw]).type_as(im_padded)
    # Id = torch.zeros([n, c, gh*gw]).type_as(im_padded)

    # for i in range(n):
    #     # Ia[i, :, :] = im_padded[i, :, x0_y0[i, 0, :]]
    #     # Ib[i, :, :] = im_padded[i, :, x0_y1[i, 0, :]]
    #     # Ic[i, :, :] = im_padded[i, :, x1_y0[i, 0, :]]
    #     # Id[i, :, :] = im_padded[i, :, x1_y1[i, 0, :]]

    #     Ia[i, :, :] = torch.index_select(im_padded[i], 1, x0_y0[i, 0, :])
    #     Ib[i, :, :] = torch.index_select(im_padded[i], 1, x0_y1[i, 0, :])
    #     Ic[i, :, :] = torch.index_select(im_padded[i], 1, x1_y0[i, 0, :])
    #     Id[i, :, :] = torch.index_select(im_padded[i], 1, x1_y1[i, 0, :])


    # 3
    x0_y0 = (x0 + y0 * padded_w).expand(-1, c, -1)  # [N, C, hw]
    x0_y1 = (x0 + y1 * padded_w).expand(-1, c, -1)
    x1_y0 = (x1 + y0 * padded_w).expand(-1, c, -1)
    x1_y1 = (x1 + y1 * padded_w).expand(-1, c, -1)

    offset = (torch.arange(c).type_as(x0_y0).reshape([c, 1]) * padded_h * padded_w) + (torch.arange(n).type_as(x0_y0).reshape([n, 1, 1]) * padded_h * padded_w * c)
    x0_y0 = x0_y0 + offset
    x0_y1 = x0_y1 + offset
    x1_y0 = x1_y0 + offset
    x1_y1 = x1_y1 + offset

    Ia = torch.take(im_padded.view(-1), x0_y0.view(-1)).view(n, c, -1)  # [N, C, hw]
    Ib = torch.take(im_padded.view(-1), x0_y1.view(-1)).view(n, c, -1)
    Ic = torch.take(im_padded.view(-1), x1_y0.view(-1)).view(n, c, -1)
    Id = torch.take(im_padded.view(-1), x1_y1.view(-1)).view(n, c, -1)

    return (Ia * wa + Ib * wb + Ic * wc + Id * wd).reshape(n, c, gh, gw)
```
