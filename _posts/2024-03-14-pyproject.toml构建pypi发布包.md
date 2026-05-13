---
layout: post
title: 'pyproject.toml构建pypi发布包'
date: 2024-03-14 02:34:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

{% raw %}
ref:
1.https://packaging.python.org/en/latest/tutorials/packaging-projects/
2.https://packaging.python.org/en/latest/guides/writing-pyproject-toml/
3.https://setuptools.pypa.io/en/latest/userguide/pyproject_config.html#
4.https://github.com/ultralytics/ultralytics/blob/main/pyproject.toml

### 1 目录结构
应该按照规范的结构创建目录
```txt
packaging_tutorial/
├── LICENSE
├── pyproject.toml
├── README.md
├── src/
│   └── example_package_YOUR_USERNAME_HERE/
│       ├── __init__.py
│       └── example.py
└── tests/
```

其中 `example_package_YOUR_USERNAME_HERE`是自己的包。我们将需要包相关的代码写在里面，一般`__init__.py`里应该还包括`__version__`和`__data__`，从构建中提取version可以通过`importlib.metadata`实现
```python
import importlib.metadata
__version__ = importlib.metadata.version("example_package_YOUR_USERNAME_HERE")
```
```python
# __init__.py

from .version import __version__, __date__
from .example import Sample

```

### 2 配置文件
这里使用`pyproject.toml`配置工程，示例为
```toml
[project]
name = "example_package_YOUR_USERNAME_HERE"
version = "0.0.1"
authors = [
  { name="Example Author", email="author@example.com" },
]
description = "A small example package"
readme = "README.md"
requires-python = ">=3.8"
dependencies = [
    "torch",
    "numpy",
    "opencv-python",
]
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
]

[project.urls]
Homepage = "https://github.com/pypa/sampleproject"
Issues = "https://github.com/pypa/sampleproject/issues"

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-data]
"example_package.utils" = ["*.yaml"]
"example_package.preweights" = ["*.pth"]

```

### 3 更新工具

```shell
# 更新工具
pip install --upgrade build
pip install twine

# 构建项目， 在pyproject.toml同目录，会生成 `.tar.gz`源码包和`.whl`安装包
python -m build

# 上传仓库， 需要先在用户路径（不是工程路径）配置`.pypirc`文件，配置`local`的仓库路径（这里可以是pypi，testpypi）、账户和token，可以在pypi注册登录查询到
twine upload -r local dist/*
```

{% endraw %}