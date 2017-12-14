---
layout:     post
title:      "Snapdragon Profiler 小记"
subtitle:   ""
date:       2017-12-14
author:     "Dafong"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - unity
---

最近新开了一个RTS项目，在做Demo的时候美术不太清楚场景地形这块怎么去弄，于是想到去参考一下其他游戏的做法，
之前有试过Intel GPA，但是GPA感觉不是很稳定，在公司的一台Nexus 6上某次成功抓到渲染数据后，后来就再也无法抓取
了，后来看到了Snapdragon Profiler，官方说是Adreno Profiler的替代版本，刚好手头的测试机也是高通的处理器，于是就下来尝个鲜。

![](/img/in-post/snapdragon-profiler.png)

## 界面
主要分为了这几个部分 DC=draw call
* Snapshot
  这个主要是提供了当前DC前的渲染结果。可以用鼠标单击某个点，会自动定位到最后的一个DC，然后会在Pixel History窗口显示出该影响该像素颜色的所有的DC历史.

* Data Explorer
  显示了所有的DC，双击某个DC后，会更新相应的Resources,Drawcall Data等窗口，单击DC可以展开，查看该DC都改变了那些GL状态。

* Resources
  这里主要显示了该快照里所有的资源包括Texture、Program(shader)，这里有两个选项All，Used，选Used会只过滤出当前双击的DC所使用的纹理和shader

* Drawcall Data
  这里就是具体的数据了，这里提供了一个导出的功能，不过只能导出obj格式的文件，obj文件是不支持顶点色，uv2等信息的，软件提供的导出属性也只有vertex normal uv1,但通常要分析游戏都会使用更多的属性，好在工具在导出的时候可以选择要将顶点的那些属性作为以上三种导出，而且obj格式也
  很好解析，所以我们可以自己通过多次导出来把自己想要的属性mapping到某个支持的字段上，然后在unity里再反向处理一下就可以完全还原顶点数据，为此我也写了一个简单的映射工具。
  ![](/img/in-post/exporter.png)

* Shader Analyzer
  在Resource选中某个program以后就可以在这里看到相应的vetex和pixel shader的代码了

* Image Preview
  在Resource选中某个纹理以后，可以在这里预览并且保存到本地，需要注意的是在安卓设备上，系统提高工的图片加载器是按照图片的原点在左上角加载的，但是在OpenGL里，原点在左下角，所以这里的图片y轴是反的，需要做一次垂直翻转才能得到原始的图片。

* Program Inspector
  这个也很有用，当我们选中某个program后，这里会显示所有的uniform值。。。

总之感觉分析功能还是很强大的，我也借助他摸清了某个游戏在场景地形方面的实现机制，甚至1:1还原了部分场景。。。感觉他也是一个非常好的学习工具。
  ![](/img/in-post/analyze-result.png)

## 注意事项
### No eligible processes found
有时会发现看不到进程，这时需要把进程杀掉，然后重新连接一下设备，然后先在profiler里Connect，然后再启动相应的进程
### Take too long time
双击drawcall的时候，有同学反映会非常慢，我这边觉得似乎还能忍受，官方在论坛里也给出了一些建议，主要是在Options里修改System的以下三项
* DepthStencil Screenshot -> False
* Full Screenshot -> False
* Highlight Screenshot -> False

### Frame delimeter
抓剑与*园的时候也发现了抓取过程会卡主，群里也有同学反映某易旗下的游戏似乎都抓不到，后来找到了参考 [https://developer.qualcomm.com/forum/qdn-forums/software/snapdragon-profiler/34240](https://developer.qualcomm.com/forum/qdn-forums/software/snapdragon-profiler/34240)
主要是更改GL Frame Delimiters，我主要是将GL Delimiter里的GL Flush设为了true，这两个游戏就都可以了，似乎高通会在以后的版本里提供UseAutoFrameDelimiter这个选项来自动选择。
