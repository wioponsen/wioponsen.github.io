---
layout: post
title: 'datasets for stereo depth'
date: 2024-05-31 09:25:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

{% raw %}
1. CRE dateset
```python
# 0,1,2,3
https://data.megengine.org.cn/research/crestereo/dataset/tree/0.tar
https://data.megengine.org.cn/research/crestereo/dataset/shapenet/0.tar
https://data.megengine.org.cn/research/crestereo/dataset/reflective/0.tar
https://data.megengine.org.cn/research/crestereo/dataset/hole/0.tar


def get_disp(disp_path):
    disp = cv2.imread(disp_path, cv2.IMREAD_UNCHANGED)
    return disp.astype(np.float32) / 32

	
```

2. Falling Things
```python
http://research.nvidia.com/publication/2018-06_Falling-Things
https://drive.google.com/file/d/1y4h9T6D9rf6dAmsRwEtfzJdcghCnI_01/view
[BT](magnet:?xt=urn:btih:5643313104D5000D183250EC341D6291FBC89554)

depth数据 
Depth along the optical axis (in 0.1 mm increments)

估算以及读txt验证 B = 600
f = 480 / tan(32°) = 768.16058349609375

disp = B*f/depth

```

3. Sceneflow
https://lmb.informatik.uni-freiburg.de/resources/datasets/SceneFlowDatasets.en.html
> The virtual imaging sensor has a size of 32.0mmx18.0mm.
Most scenes use a virtual focal length of 35.0mm. For those scenes, the virtual camera intrinsics matrix is given by
fx=1050.0	0.0	cx=479.5
0.0	fy=1050.0	cy=269.5
0.0	0.0	1.0
where (fx,fy) are focal lengths and (cx,cy) denotes the principal point.
Some scenes in the Driving subset use a virtual focal length of 15.0mm (the directory structure describes this clearly). For those scenes, the intrinsics matrix is given by
fx=450.0	0.0	cx=479.5
0.0	fy=450.0	cy=269.5
0.0	0.0	1.0
Please note that due to Blender's coordinate system convention (see below), the focal length values (fx,fy) really should be negative numbers. Here we list the positive numbers because in practise this catch is only important when working on the raw 3D data.

fx = 1050, B = 1.0

```python

https://lmb.informatik.uni-freiburg.de/data/SceneFlowDatasets_CVPR16/Release_april16/data/FlyingThings3D/raw_data/flyingthings3d__frames_cleanpass.tar
https://lmb.informatik.uni-freiburg.de/data/SceneFlowDatasets_CVPR16/Release_april16/data/FlyingThings3D/derived_data/flyingthings3d__disparity.tar.bz2
```

4. Sintel
```python
http://sintel.is.tue.mpg.de/downloads

disp = cv2.imread("disp..", cv2.IMREAD_UNCHANGED).astype(np.float64)
disp_float = disp[:, :, 2] * 4 + disp[:, :, 1] / (2 ** 6) + disp[:, :, 0] / (2 ** 14)

baseline = 10 # cm
fov = 72 
fx = 687 
```

5.others
[InStereo2K] https://github.com/YuhuaXu/StereoDataset
disp = u16 / 100.


read/write pfm file: https://lmb.informatik.uni-freiburg.de/resources/datasets/IO.py
##### Read pfm files
```python
import re
import sys
import numpy as np

def read_pfm(filename):
    with open(filename, 'rb') as file:
        # header
        header = file.readline().rstrip()
        if (sys.version[0]) == '3':
            header = header.decode('utf-8')
        if header == 'PF':
            color = True
        elif header == 'Pf':
            color = False
        else:
            raise Exception('Not a PFM file.')

        # width height
        if (sys.version[0]) == '3':
            dim_match = re.match(r'^(\d+)\s(\d+)\s$', file.readline().decode('utf-8'))
        else:
            dim_match = re.match(r'^(\d+)\s(\d+)\s$', file.readline())
        if dim_match:
            width, height = map(int, dim_match.groups())
        else:
            raise Exception('Malformed PFM header.')

        # scale
        if (sys.version[0]) == '3':
            scale = float(file.readline().rstrip().decode('utf-8'))
        else:
            scale = float(file.readline().rstrip())

        # endian
        if scale < 0:  # little-endian
            endian = '<'
            scale = -scale
        else:
            endian = '>'  # big-endian

        # data
        data = np.fromfile(file, endian + 'f')
    shape = (height, width, 3) if color else (height, width)

    data = np.reshape(data, shape)
    data = np.flipud(data)
    return data, scale

```

{% endraw %}