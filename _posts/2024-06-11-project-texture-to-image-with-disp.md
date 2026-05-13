---
title: 'project texture to image with disp'
date: 2024-06-11 06:35:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

#### 视差投影推理

>  已知相机视差图和其x方向某点的纹理图，求纹理投影到相机图的对应关系。

对于相机，其成像在等效焦距平面上的空间坐标可以得到，为
$$
p = (x-\frac{w}{2}, y-\frac{h}{2}, f_0)
$$
考虑到深度与视差的关系：
$$
depth = \frac{B*f_0}{disp}
$$
可以推导出实际像素对应的空间点坐标为
$$
s = \frac{depth}{f_0} = \frac{B}{disp}  \\
P = ((x-\frac{w}{2})*s, (y-\frac{h}{2})*s, \frac{Bf_0}{disp})
$$
而假设偏移点位置在 
$$
B = (kB, 0, 0)
$$
那么可以获得射线BP
$$
BP = ((x-\frac{w}{2})*s - kB, (y-\frac{h}{2})*s, \frac{Bf_0}{disp})
$$
将射线投影到$z=f_{texture}$平面上
$$
BP_{z=f_{texture}} = ((x - \frac{w}{2})-k*disp, (y - \frac{h}{2}), f_0) *\frac{f_{texture}}{f_0}
$$
令$scale=\frac{f_{texture}}{f_0}$,纹理图宽高为$tw, th$ ,那么相机像素坐标到纹理图的映射关系应该是
$$
(x, y) : (((x - \frac{w}{2})-k*disp)*scale+\frac{tw}{2}, (y - \frac{h}{2})*scale+\frac{th}{2})
$$


```python
def project_texture(texture, disp, loc, project_scale, shift_xy=(0, 0)):
    _h, _w = disp.shape[0:2]
    _dh, _dw = texture.shape[0:2]
    disp = disp.astype(np.float32)
    xx_grid = np.arange(_w).reshape([1, -1]).repeat(_h, axis=0).astype(np.float32) - _w // 2
    yy_grid = np.arange(_h).reshape([-1, 1]).repeat(_w, axis=1).astype(np.float32) - _h // 2
    vector_xx = xx_grid - loc * disp
    vector_yy = yy_grid
    map_xx = vector_xx * project_scale + _dw // 2 + shift_xy[0]
    map_yy = vector_yy * project_scale + _dh // 2 + shift_xy[1]

    warp_texture = cv2.remap(texture, map_xx, map_yy, interpolation=cv2.INTER_LINEAR)
    return warp_texture


def refine_texture(texture, disp, board=2):
    edges = cv2.Canny(image=(disp.astype(np.float32)).astype(np.uint8), threshold1=7, threshold2=50)
    for i, j in np.argwhere(edges > 0):
        u = np.clip(i - board, 0, edges.shape[0] - 1)
        d = np.clip(i + board, 0, edges.shape[0] - 1)
        l = np.clip(j - board, 0, edges.shape[1] - 1)
        r = np.clip(j + board, 0, edges.shape[1] - 1)
        texture[i, j] = np.mean(texture[u:d, l:r])

    return texture, edges


def read_process_once(left_path_name, texure_name):
    left_name = left_path_name
    left_disp_name = left_path_name[:-4]+'_disp.png'
    right_name = left_path_name[:-9]+'_right.png'
    right_disp_name = left_path_name[:-9]+'_right_disp.png'

    assert os.path.isfile(right_disp_name), "No right DISPARITY file!"

    left = cv2.imread(left_name, 0)
    right = cv2.imread(right_name, 0)

    left_disp = cv2.imread(left_disp_name, cv2.IMREAD_UNCHANGED)
    right_disp = cv2.imread(right_disp_name, cv2.IMREAD_UNCHANGED)

    left_fp32 = left.astype(np.float32) / 255.0
    right_fp32 = right.astype(np.float32) / 255.0
    left_fp32 = np.power(left_fp32, 2.5)
    right_fp32 = np.power(right_fp32, 2.5)

    left_disp_fp32 = left_disp.astype(np.float32) / 128.
    right_disp_fp32 = right_disp.astype(np.float32) / 128.

    texture = cv2.imread(texure_name, cv2.IMREAD_UNCHANGED).astype(np.float32) / 1023.
    project_scale = np.random.uniform(0.9, 1.1)
    project_scale = project_scale if texture.shape[0] >= 720 else (project_scale*0.5)

    loc = np.random.uniform(0.45, 0.75)  # R1.5~2.1 / 4.8  == [0.52, 0.69]
    shift_x = np.random.uniform(-20, 20)
    shift_y = np.random.uniform(-10, 10)
    left_texture = project_texture(texture, left_disp_fp32, loc, project_scale, shift_xy=(shift_x, shift_y))
    left_texture, left_edge = refine_texture(left_texture, left_disp_fp32)
    right_texture = project_texture(texture, right_disp_fp32, loc - 1., project_scale, shift_xy=(shift_x, shift_y))
    right_texture, right_edge = refine_texture(right_texture, right_disp_fp32)

    weight = np.random.uniform(0.35, 0.65)
    left_fp32 = left_fp32 * weight + left_texture * (1 - weight)
    right_fp32 = right_fp32 * weight + right_texture * (1 - weight)

    fuse_left = (left_fp32.clip(0., 1.) * 1023).astype(np.uint16)
    fuse_right = (right_fp32.clip(0., 1.) * 1023).astype(np.uint16)

    stereo = np.hstack((fuse_right, fuse_left))
    return stereo, left_disp, right_disp


```