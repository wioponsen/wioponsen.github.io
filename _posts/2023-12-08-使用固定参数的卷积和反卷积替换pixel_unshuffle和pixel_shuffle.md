---
layout: post
title: '使用固定参数的卷积和反卷积替换pixel_unshuffle和pixel_shuffle'
date: 2023-12-08 07:48:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

{% raw %}
pixel_unshuffle 和 pixel_shuffle 在导出ONNX的时候比较麻烦；
由于该op需要操作内存， 可以减小运算量，所以用的很多，但是内存操作在设备端可能是低效的；
而conv2d 和conv_transpose2d 在设备端的优化效率很高，往往会有更好性能

替换代码
```python
import numpy as np
import torch
import torch.nn as nn
import torch.nn.functional as F


class Replace_Pixel_Unshuffle(nn.Module):
    # space2depth
    def __init__(self, block_size):
        super(Replace_Pixel_Unshuffle, self).__init__()
        self.block_size = block_size
        self.weight = None

    def create_weight_np(self, block, cin):
        weight = np.zeros([cin * block * block, cin, block, block])
        # for pytorch, cxbxb

        for i in range(cin):
            for j in range(block):
                for k in range(block):
                    weight[i * block * block + j * block + k, i, j, k] = 1.0

        return np.array(weight).astype(np.float32)

    def forward(self, x):
        if self.weight is None:
            cin = x.shape[1]
            self.weight = torch.from_numpy(self.create_weight_np(self.block_size, cin)).to(x.device)

        out = F.conv2d(x, self.weight, None, stride=self.block_size)

        return out


class Replace_Pixel_Shuffle(nn.Module):
    # depth2space
    def __init__(self, block_size):
        super(Replace_Pixel_Shuffle, self).__init__()
        self.block_size = block_size
        self.weight = None

    def create_weight_np(self, block, trans_cin):
        # [cin, cout, k, k]
        cin = trans_cin // block // block
        weight = np.zeros([cin * block * block, cin, block, block])
        # for pytorch, cxbxb

        for i in range(cin):
            for j in range(block):
                for k in range(block):
                    weight[i * block * block + j * block + k, i, j, k] = 1.0

        return np.array(weight).astype(np.float32)

    def forward(self, x):
        if self.weight is None:
            cin = x.shape[1]
            self.weight = torch.from_numpy(self.create_weight_np(self.block_size, cin)).to(x.device)

        out = F.conv_transpose2d(x, self.weight, None, stride=self.block_size)

        return out


class TestModel(nn.Module):
    def __init__(self):
        super(TestModel, self).__init__()
        self.down = Replace_Pixel_Unshuffle(2)
        self.up = Replace_Pixel_Shuffle(2)

        self.pt_down = nn.PixelUnshuffle(2)
        self.pt_up = nn.PixelShuffle(2)

    def forward(self, x):
        r1 = self.down(x)
        r2 = self.pt_down(x)
        d1 = r1 - r2

        r1 = self.up(r1)
        r2 = self.pt_up(r2)
        d2 = r1 - r2
        return d1, d2


if __name__ == '__main__':
    print('Start')
    inputs = torch.rand([1, 8, 64, 64])
    net = TestModel()
    diff1, diff2 = net(inputs)
    print(diff1.min(), diff1.max())
    print(diff2.min(), diff2.max())
    print('Done')


```

另外有些情况下，不希望onnx中不同tensor有参数复用的情况，可以在简化的时候加上
```python
model_simp, check = simplify(model, perform_optimization=True, skipped_optimizers=['eliminate_duplicate_initializer'])
```

对于初始化同权重的tensor，目前测试， pytorch1.x直接导出onnx的时候是两个独立， 而在pytorch2.0中，会多一个identity将两个参数指向同一个
onnx-sim默认也会复用参数，使用上述指令即可避免
{% endraw %}