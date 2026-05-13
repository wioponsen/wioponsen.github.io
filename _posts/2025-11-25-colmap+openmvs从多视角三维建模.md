---
layout: post
title: 'colmap+openMVS从多视角三维建模'
date: 2025-11-25 07:13:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

{% raw %}

COLMAP部分建图，将图片拷贝到 `文件夹/images` ,然后依次执行

```bat
set WORK_PATH=test

colmap database_creator --database_path %WORK_PATH%/database.db

MD %WORK_PATH%\sparse
MD %WORK_PATH%\dense

colmap feature_extractor --database_path %WORK_PATH%/database.db --image_path %WORK_PATH%/images --ImageReader.camera_model PINHOLE --ImageReader.single_camera 1 --ImageReader.camera_params "1063.15,1063.15,637.2,480.5" --SiftExtraction.upright 1

REM colmap feature_extractor --database_path %WORK_PATH%/database.db --image_path %WORK_PATH%/images --ImageReader.camera_model PINHOLE --ImageReader.single_camera 1 --ImageReader.camera_params "529.918,529.918,320,240" --SiftExtraction.upright 1 --SiftExtraction.contrast_threshold 0.01 --SiftExtraction.edge_threshold 10 --SiftExtraction.max_num_features 10000

colmap exhaustive_matcher --database_path %WORK_PATH%/database.db --SiftMatching.guided_matching 1 

colmap mapper --database_path %WORK_PATH%/database.db --image_path %WORK_PATH%/images --output_path %WORK_PATH%/sparse

colmap model_converter --input_path %WORK_PATH%/sparse/0 --output_path %WORK_PATH%/sparse/0 --output_type TXT

colmap image_undistorter --image_path %WORK_PATH%/images  --input_path %WORK_PATH%/sparse/0  --output_path %WORK_PATH%/dense  --output_type COLMAP

REM colmap patch_match_stereo --workspace_path %WORK_PATH%/dense  --workspace_format COLMAP  --PatchMatchStereo.geom_consistency true

colmap patch_match_stereo --workspace_path %WORK_PATH%/dense  --workspace_format COLMAP  --PatchMatchStereo.geom_consistency true --PatchMatchStereo.depth_min 0.5 --PatchMatchStereo.depth_max 10.0

colmap stereo_fusion --workspace_path %WORK_PATH%/dense --workspace_format COLMAP --output_path %WORK_PATH%/dense/fused.ply --StereoFusion.max_depth_error 0.02 --StereoFusion.min_num_pixels 5

```

其中的一些参数：
```bat
1. 创建数据库，并提取图像特征：
colmap feature_extractor
可以指定相机的参数
--ImageReader.camera_model PINHOLE --ImageReader.single_camera 1 --ImageReader.camera_params "1065.56,1065.56,639.66,479.67"
启用upright SIFT加速处理。如果图像之间没有较大的旋转（<15度），那么开启优化，可以不计算主方向，加速
--SiftExtraction.upright 1 
有些情况可以增加特征点，比如主体占比较小。降低阈值提取更多弱特征；减少边缘特征过滤；增加最大特征点数
--SiftExtraction.contrast_threshold 0.01 --SiftExtraction.edge_threshold 10 --SiftExtraction.max_num_features 10000



2. 相机参数和位姿读取
可以将`.bin`转为`.txt`
colmap model_converter --input_path %WORK_PATH%/sparse/0 --output_path %WORK_PATH%/sparse/0 --output_type TXT


```

openMVS下进行三维重建和贴图：
```
set WORK_PATH=test

# 将colmap转化为openMVS格式
InterfaceCOLMAP -i %WORK_PATH%/dense -o %WORK_PATH%/dense/scene.mvs --image-folder %WORK_PATH%/dense/images

cd %WORK_PATH%\dense

# 重建三维模型，会生成mesh.ply， mesh.mvs位姿参数没有改变不会生成新的mvs
ReconstructMesh -i scene.mvs -o mesh.mvs

# 用图像进行贴图，这里mesh.ply可以手动裁剪，可以认为指定
TextureMesh -i scene.mvs -m mesh-sim.ply -o texure_model.obj --export-type obj

```

其他的一些：
从`image.txt`和`cameras.txt`可以读取内外参：
```python
import os
import numpy as np
from scipy.spatial.transform import Rotation as R

def read_cameras(txt):
    with open(txt, 'r') as f:
        cameras_lines = f.readlines()

    camera_map = {}

    for line in cameras_lines:
        if line.startswith('#'):
            continue
        else:
            line_slipt = line.split()
            fx = float(line_slipt[4])
            fy = float(line_slipt[5])
            cx = float(line_slipt[6])
            cy = float(line_slipt[7])
            camera_map[line_slipt[0]] = np.array([[fx, 0, cx], [0, fy, cy], [0,0,1]]).astype(np.float32)

    return camera_map


def read_images(image_txt, cameras_txt):
    map_camera = read_cameras(cameras_txt)

    with open(image_txt, 'r') as f:
        images_lines = f.readlines()

    for line in images_lines:
        if line.startswith('#'):
            continue
        else:
            line_slipt = line.split()
            if len(line_slipt) == 10:
                name = line_slipt[-1]
                camera_id = line_slipt[-2]
                QW, QX, QY, QZ, TX, TY, TZ = line_slipt[1:8]
                QW, QX, QY, QZ, TX, TY, TZ = float(QW), float(QX), float(QY), float(QZ), float(TX), float(TY), float(TZ)
                rot_mat = R.from_quat([QX, QY, QZ, QW]).as_matrix()
                trans_mat = np.array([TX, TY, TZ]).reshape([3, 1])

                trans4x4 = np.eye(4, 4, dtype=np.float32)
                trans4x4[0:3, 0:3] = rot_mat[:, :]
                trans4x4[0:3, 3:4] = trans_mat[:, :]


                np.savetxt(name.replace('rgb', 'tran')[:-4]+'.txt', trans4x4, fmt='%.3f')
                np.savetxt(name.replace('rgb', 'iparam')[:-4] + '.txt', map_camera[camera_id], fmt='%.5f')


if __name__ == '__main__':
    read_images('images.txt', 'cameras.txt')


```

{% endraw %}