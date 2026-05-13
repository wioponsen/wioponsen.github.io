---
title: 'open3d读取ply和obj的差异'
date: 2026-05-06 09:39:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

原创，转载注明出处：https://www.cnblogs.com/wioponsen/p/19981866

如果发现open3d读取ply和obj，后续处理顶点，发现差异很大，可以试试进行如下操作：
```
    mesh = o3d.io.read_triangle_mesh(obj_name, enable_post_processing=False)

    # obj -> ply， 合并顶点
    mesh = mesh.remove_duplicated_vertices()
    mesh = mesh.remove_non_manifold_edges()
    # 处理纹理文件
    tex_img = cv2.imread(obj_name[:-4]+"_map_Kd.png")
    tex_rgb = np.ascontiguousarray(tex_img[::-1, :, ::-1])
    mesh.textures = [o3d.geometry.Image(tex_rgb)]
    print(f"简化后顶点数: {len(mesh.vertices)}")

```

问题原因：
读取ply: PLY 倾向于“顶点属性化”，即一个点带一个颜色，open3d读取出的点数和文件中点数一致；
读取obj: OBJ 读取的时候，因为带纹理，其顶点数量可能比原始的点更多，三角面中每个点可能对应多个uv, open3d读取出来的点数比原始要多;

所以如果有自定义的会修改顶点坐标的操作，在操作obj之后，会出现索引偏移，导致效果出错，建议先检查顶点数量：
```
print(len(mesh.vertices))
```