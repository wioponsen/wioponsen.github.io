---
layout: post
title: '使用pybind11将C/C++代码封包在python中使用'
date: 2026-04-29 03:37:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

{% raw %}
内容原创，转载注明出处：https://www.cnblogs.com/wioponsen/p/19949084

## 0 目标
将自己的C/C++库构建成python能调用的包

## 1 方案
1. 可以通过 `CMakeLists.txt` 编写cmake项目，然后指定python环境，pybind11环境实现
2. 【推荐，更简单】可以直接通过配置 `setup.py` 在python环境中直接将代码封包

## 2 准备python接口文件，pybind_my_package.cpp
从C/C++需要对接口进行处理，从而确定python中能够使用的接口（相当于动态库导出哪些函数、类、结构定义等）和数据类型。
尽量使用标准结构，这样能够自动处理；对于复杂结构，可以使用Eigen库，python对该库的自动处理很完善，能自动转成numpy等格式

官方文档已经很详细，https://pybind11.readthedocs.io/en/stable/
以下是我的例子：
```C++
// pybind_my_package.cpp
#pragma once

#include <pybind11/pybind11.h>
#include <pybind11/stl.h>  // 标准类型支持
#include <pybind11/eigen.h>  // Eigen支持，支持向量、矩阵，tensor等

#include <vector>
#include <string>

#include "my_package.hpp" // 原C++接口头文件

namespace py = pybind11;

PYBIND11_MODULE(my_package, m) {  // 这里为导出包取名 my_package， m是当前module，可以将多个多种类型的接口包括进来
    m.doc() = "Measure Breast Python Binding";  // python doc 说明

    py::class_<MyPackageClass>(m, "MyPackageClass")  // class注册，<>内是原数据结构， " "内是python中的模块名字
      .def(py::init<const std::vector<Eigen::Vector3d>&, const std::vector<Eigen::Vector3i>&, const std::vector<Eigen::Vector3d>&, const std::vector<std::unordered_set<int>>& >()) // py::init<>（）是构造函数
      .def("test_func", &MyPackageClass::testFunc) // 成员函数注册， 无参数
      .def("set_radius", &MyPackageClass::setRadius, 
                    py::arg("radius"), py::arg("is_left")； // 成员函数注册， 有参数，参数使用 py::arg("") 指定

    py::enum_<Shape>(m, "Shape") // 枚举类型
        .value("Round", Shape::Round)
        .export_values();

    m.def("shape_to_string", &ShapeToString,  // 静态函数
          "Convert Shape to string");

    py::class_<Params>(m, "Params")  //结构体
        .def(py::init<>()) // 默认构造
        .def(py::init<int, int, double, double, std::vector<int>, std::vector<int>>(), // 自定义构造
              py::arg("left"), py::arg("right"),
              py::arg("radius_left"), py::arg("radius_right"),
              py::arg("contours_left"), py::arg("contours_right"))
        .def_readwrite("left", &Params::left)  // 成员变量
        .def_readwrite("right", &Params::right)
        .def_readwrite("radius_left", &Params::radius_left)
        .def_readwrite("radius_right", &Params::radius_right)
        .def_readwrite("contours_left", &Params::contours_left)
        .def_readwrite("contours_right", &Params::contours_right);
}


```




## 3 CMakeLists.txt 方案
需要查找 `python` 和 `pybind11`
```cmake
# set(Python3_ROOT "C:/Python314")
# set(CONDA_ENV_PATH "C:/MySoftware/miniconda3")
find_package(Python3 COMPONENTS Interpreter Development REQUIRED)

# find_package(pybind11 REQUIRED)
include(FetchContent)
FetchContent_Declare(pybind11 GIT_REPOSITORY https://github.com/pybind/pybind11.git GIT_TAG v3.0.4)
FetchContent_MakeAvailable(pybind11)
```
其中`pybind11`可以直接在python中安装，而不选择从github更新
```cmd
# 安装global保证安装路径中有cmake配置 `site-packages\pybind11\share\cmake\pybind11\*.cmake`
# 这样在cmake的时候，可以手动定位到该目录

pip install pybind11[global]

```

以下是例子，也可以拆分使用 `add_subdirectory` 支持多组导出
```cmake
cmake_minimum_required(VERSION 3.15)
project(my_package)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ============ 公共依赖 ============
# set(Python3_ROOT "C:/Python314")
set(CONDA_ENV_PATH "C:/MySoftware/miniconda3")
find_package(Python3 COMPONENTS Interpreter Development REQUIRED)
#find_package(Numpy REQUIRED)

# ============ pybind11 ============
# find_package(pybind11 REQUIRED)
include(FetchContent)
FetchContent_Declare(pybind11 GIT_REPOSITORY https://github.com/pybind/pybind11.git GIT_TAG v3.0.4)
FetchContent_MakeAvailable(pybind11)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_compile_options(-O0 -g)
else()
    add_compile_options(-O3)
endif()

# ============ 依赖路径 ============
set(INCLUDE_DIR ${CMAKE_SOURCE_DIR}/../src/include)
set(EIGEN_INCLUDE ${INCLUDE_DIR}/Eigen)
set(IGL_INCLUDE ${INCLUDE_DIR}/igl)


# ============ 创建 Python 模块 ============
pybind11_add_module(my_package 
    pybind_my_package.cpp
)

# ============ 头文件路径 ============
target_include_directories(my_package PRIVATE
    ${CMAKE_SOURCE_DIR}
    ${INCLUDE_DIR}
    ${EIGEN_INCLUDE}
    ${IGL_INCLUDE}
    ${Python3_NumPy_INCLUDE_DIRS}
)

# ============ 编译定义（libigl header-only）===========
target_compile_definitions(my_package PRIVATE
    PYBIND11_DETAIL_ABI
)

# install(TARGETS my_package
#         LIBRARY DESTINATION "${CMAKE_INSTALL_PREFIX}"
#         RUNTIME DESTINATION "${CMAKE_INSTALL_PREFIX}")

# ============ 安装（可选）===========
# install(TARGETS my_package
#         LIBRARY DESTINATION ${Python3_SITEARCH})
```


## 4 setup.py 方案
直接编写 `setup.py`文件

```python
import os
from setuptools import setup, Extension
import numpy as np
import pybind11

# OpenCV 配置
OPENCV_DIR = "D:/opencv480/build"
OPENCV_LIB_NAME = "opencv_world480"
OPENCV_LIB_DIR = f"{OPENCV_DIR}/x64/vc16/lib"
OPENCV_DLL_DIR = f"{OPENCV_DIR}/x64/vc16/bin"

## 该代码为写的文件拷贝功能，可以在文件修改的时候从主目录中更新拷贝，不需要则注释
import shutil
import filecmp
def file_copy(src, dst):
    if not os.path.exists(dst) or not filecmp.cmp(src, dst, shallow=False):
        shutil.copy2(src, dst)

file_copy("../my_package.hpp", "my_package.hpp")
file_copy(os.path.join(OPENCV_DLL_DIR, OPENCV_LIB_NAME + ".dll"), OPENCV_LIB_NAME + ".dll")

## 将pybind11接口的cpp包含进来
sources = ["pybind_my_package.cpp"]

## 可以指定编译参数
extra_compile_args = ["/std:c++17"]
extra_link_args = []

ext_modules = [
    Extension(
        "my_package", # 指定模块名称， 与pybind_my_package.cpp中一致
        sources=sources,
        include_dirs=[
			os.path.dirname(__file__),
			os.path.join(os.path.dirname(__file__), "../../src/include"),
            os.path.join(os.path.dirname(__file__), "../../src/include/Eigen"),
            f"{OPENCV_DIR}/include",
            pybind11.get_include(),
            np.get_include(),
        ],
        library_dirs=[
            OPENCV_LIB_DIR,
        ],
        libraries=[
            OPENCV_LIB_NAME,
        ],
		extra_compile_args=extra_compile_args,
		extra_link_args=extra_link_args,
    )
]

## 需要一起打包的文件可以归总在一起，写进 data_files
dlls = [os.path.join(os.path.dirname(__file__), "opencv_world480.dll")]


setup(
    name="my_package",
    version="0.1.0",
    author="your_name",
    description="Your doc about Python Binding",
    ext_modules=ext_modules,
	bdist_wheel={
        "universal": False, 
    },
	data_files=[("", dlls)],
	include_package_data=True,
)

```

编写好`setup.py`后，可以进行编译：
```cmd
# 如果使用特定conda的python环境
conda activate base 

# 编译
python setup.py build_ext --inplace
# 安装
# 拷贝到 site-packages 安装
pip install .
# 链接到当前目录，作为安装
pip install -e .

## 直接编译打包 whl 包
python setup.py bdist_wheel
# 安装
pip install dist/*.whl


```

#### 4.1 补充
如果用cmake生成了pyd文件，可以在次基础上用 `setup.py`打包

将生成的pyd和库拷贝到新的文件夹 `my_package` 下， 然后在外面编写如下 `setup.py` 文件
```python
from setuptools import setup, find_packages

setup(
    name="my_package",
    version="0.1.0",
    packages=find_packages(),
    package_data={
        "my_package": ["*.pyd", "*.dll"],
    },
)

```
最后使用
```cmd
python setup.py bdist_wheel
```
即可打包whl

## 5 需要注意
1. C/C++代码尽量使用标准结构作为函数的输入和输出，对应pybind11的stl; 需要使用矩阵等，尽量使用eigen库，对应pybind11的eigen
2. 安装包的时候如果有dll依赖库等，需要注意是否安装到了 site-packages 目录，如果python安装在c盘，是有可能出现权限问题导致拷贝不成功，这时候就需要手动拷贝

{% endraw %}