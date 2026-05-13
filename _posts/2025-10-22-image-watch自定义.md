---
title: 'image watch自定义'
date: 2025-10-22 02:38:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

安装`image watch`模块除了可以查看OpenCV，也可以查看自定义的图像格式。

可以参考的官方文档： https://learn.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2015/debugger/image-watch/image-watch-reference?view=vs-2015

可以从 `C:\Users\USERNAME\AppData\Local\Microsoft\VisualStudio\17.0_6e3f40f8\Extensions\a14eewtt.z12` 文件夹看到`OpenCV`的配置文件 `ImageWatchOpenCV.natvis` 进行对照参考
```C++
<?xml version="1.0" encoding="utf-8"?>
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010">
  <UIVisualizer ServiceId="{A452AFEA-3DF6-46BB-9177-C0B08F318025}" Id="1" MenuName="Add to Image Watch"/>
  

<!-- OpenCV cv::Mat -->

  <Type Name="cv::Mat">
    <AlternativeType Name="cv::Mat_&lt;*&gt;"/>
    <UIVisualizer ServiceId="{A452AFEA-3DF6-46BB-9177-C0B08F318025}" Id="1" />
  </Type>

  <Type Name="cv::Mat">
    <AlternativeType Name="cv::Mat_&lt;*&gt;"/>
    <DisplayString Condition="(flags&amp;7)==0">{{UINT8, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <DisplayString Condition="(flags&amp;7)==1">{{INT8, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <DisplayString Condition="(flags&amp;7)==2">{{UINT16, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <DisplayString Condition="(flags&amp;7)==3">{{INT16, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <DisplayString Condition="(flags&amp;7)==4">{{INT32, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <DisplayString Condition="(flags&amp;7)==5">{{FLOAT32, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <DisplayString Condition="(flags&amp;7)==6">{{FLOAT64, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <DisplayString Condition="(flags&amp;7)==7">{{USER, {((flags&amp;0xfff)&gt;&gt;3)+1} x {cols} x {rows}}}</DisplayString>
    <Expand>
      <Synthetic Name="[type]" Condition="(flags&amp;7)==0">
        <DisplayString>UINT8</DisplayString>
      </Synthetic>
      <Synthetic Name="[type]" Condition="(flags&amp;7)==1">
        <DisplayString>INT8</DisplayString>
      </Synthetic>
      <Synthetic Name="[type]" Condition="(flags&amp;7)==2">
        <DisplayString>UINT16</DisplayString>
      </Synthetic>
      <Synthetic Name="[type]" Condition="(flags&amp;7)==3">
        <DisplayString>INT16</DisplayString>
      </Synthetic>
      <Synthetic Name="[type]" Condition="(flags&amp;7)==4">
        <DisplayString>INT32</DisplayString>
      </Synthetic>
      <Synthetic Name="[type]" Condition="(flags&amp;7)==5">
        <DisplayString>FLOAT32</DisplayString>
      </Synthetic>
      <Synthetic Name="[type]" Condition="(flags&amp;7)==6">
        <DisplayString>FLOAT64</DisplayString>
      </Synthetic>
      <Item Name="[channels]">((flags&amp;0xfff)&gt;&gt;3)+1</Item>
      <Item Name="[width]">cols</Item>
      <Item Name="[height]">rows</Item>
      <Item Name="[data]">data</Item>
      <Item Name="[stride]">step.p[0]</Item>
    </Expand>
  </Type>
</AutoVisualizer>
```

然后结合官方示例文档：
```C++
struct My8BitRGBImage 
{ 
    unsigned int ncols; 
    unsigned int nrows; 
    unsigned char* data; 
};

--------------------
<?xml version="1.0" encoding="utf-8"?> 
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010"> 
  <UIVisualizer ServiceId="{A452AFEA-3DF6-46BB-9177-C0B08F318025}" Id="1"  
                MenuName="Add to Image Watch"/> 
  <Type Name="My8BitRGBImage"> 
    <UIVisualizer ServiceId="{A452AFEA-3DF6-46BB-9177-C0B08F318025}" Id="1" /> 
  </Type> 
  <Type Name="My8BitRGBImage"> 
    <Expand> 
      <Synthetic Name="[type]"> 
        <DisplayString>UINT8</DisplayString> 
      </Synthetic> 
      <Synthetic Name="[channels]"> 
        <DisplayString>RGB</DisplayString> 
      </Synthetic> 
      <Item Name="[width]">ncols</Item> 
      <Item Name="[height]">nrows</Item> 
      <Item Name="[data]">data</Item> 
      <Item Name="[stride]">ncols*3</Item> 
    </Expand> 
  </Type>   
</AutoVisualizer>
```

所以`Type Name`设置为`class`类名，图像内部需要设置 `type|channels|width|height|data|stride`等参数。
如果要对模板类进行处理（模版类的中括号<(`&lt;`)   >(`*&gt;`)  &（`&amp;`）），需要针对类型去做type区分， 这里的`Condition=`就相当于 `switch-case`,type支持的数据类型从官方文档可以查到：
```C++
image Watch supports these channel types:
INT8, UINT8
INT16, UINT16
INT32
FLOAT16
FLOAT32
FLOAT64

defines the data layout，channels:
RG, UV
RGB, BGR, YUV
RGBA, BGRA


1 channel image: grayscale or pseudo color depending on the current viewer setting.
2 channel image: red/green
3 channel image: B, G, R
4 channel image: B, G, R, Alpha, or ignore Alpha depending on current viewer setting.
5 or more channels: map first three channels to B, G, R, and ignore remaining channels

A number of special YUV formats are supported as well. In this case, the format string also defines the data layout. For a detailed description of YUV formats, see YUV pixel formats.
NV12 (two planes: one Y plane, one packed UV plane, subsampled by 2 in both dimensions)
YV12 (three planes: one Y plane, one packed U and V plane each, both subsampled by 2 in both dimensions)
IYUV (same as YV12 but with U and V planes switched)
YUY2 (single plane, interleaved two-channel format: Y in the first channel; U and V subsampled by 2 horizontally and stored alternating in the second channel)
```
官方示例：
```C++
template <typename T> 
struct MyGenericImage 
{	 
    unsigned int ncols; 
    unsigned int nrows; 
    unsigned int nchannels; 

    T* data; 
};
------------------------
<?xml version="1.0" encoding="utf-8"?> 
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010"> 
  <UIVisualizer ServiceId="{A452AFEA-3DF6-46BB-9177-C0B08F318025}" Id="1"  
                MenuName="Add to Image Watch"/> 
  <Type Name="MyGenericImage&lt;*&gt;"> 
    <UIVisualizer ServiceId="{A452AFEA-3DF6-46BB-9177-C0B08F318025}" Id="1" /> 
  </Type> 
  <Type Name="MyGenericImage&lt;*&gt;"> 
    <Expand> 
      <Synthetic Name="[type]" Condition='strcmp("unsigned char", "$T1") == 0'> 
        <DisplayString>UINT8</DisplayString> 
      </Synthetic> 
      <Synthetic Name="[type]" Condition='strcmp("float", "$T1") == 0'> 
        <DisplayString>FLOAT32</DisplayString> 
      </Synthetic> 
      <Item Name="[channels]">nchannels</Item> 
      <Item Name="[width]">ncols</Item> 
      <Item Name="[height]">nrows</Item> 
      <Item Name="[data]">data</Item> 
      <Item Name="[stride]">ncols*nchannels*sizeof($T1)</Item> 
    </Expand> 
  </Type> 
</AutoVisualizer>
```
Note the “*” wildcard syntax, the Condition attribute, and the $T1 reference to the template argument. 
Intrinsic functions(strcmp,sizeof等): `https://learn.microsoft.com/en-us/visualstudio/debugger/expressions-in-the-debugger?view=vs-2022`
编写好`.natvis` 并保存在
```
C:\Users\%USER_NAME%\Documents\Visual Studio 2022\Visualizers

安装位置应该也是可以的：
c:\users\%USER_NAME%\appdata\local\microsoft\visualstudio\17.0_6e3f40f8\extensions\a14eewtt.z12
%INSTALL_PATH%\VS2022\VS2022\Common7\Packages\Debugger\Visualizers
C:\Users\%USER_NAME%\Documents\Visual Studio 2022\Visualizers
```
即可支持自定义图像格式。
