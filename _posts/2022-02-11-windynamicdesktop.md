---
layout: post
title: "基于WinDynamicDesktop制作冬奥二十四节气主题"
subtitle: "Making Winter Olympics Twenty-Four Solar Terms Theme Based on WinDynamicDesktop"
author: "PYQ"
header-img: "img/post-bg-theme.png"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - WinDynamicDesktop
---

## Ⅰ. 前言

2022北京冬奥会开幕式为全国人民上演了一场精彩纷呈的艺术盛宴，展现了浓浓的中国元素、满满的民族自信和炫丽的科技风采。其中使用中华民族农耕文明的结晶二十四节气进行倒计时给我留下了极深的印象，将祖国的大美江山、二十四节气和古典诗词相结合。

看完短片就油发了使用节气图片作为壁纸的想法。二十四节气正好二十四张图片，对应二十四小时，有没有什么方法使电脑每小时换一张节气图片，同时每小时对应相应的节气呢？

第一个想法是使用Wallpaper Engine实现，但没有制作Wallpaper Engine主题的经验，同时较大的主题会消耗较多的内存和显存，对配置较低的电脑不太友好，遂放弃。

第二个想法则是使用目前正在使用的壁纸软件WinDynamicDesktop，WinDynamicDesktop将macOS的动态桌面功能移植到Windows 中，它使用电脑的位置来确定日出和日落的时间，并根据一天的时间更改桌面壁纸。同时这款软件由于是通过更改Windows设置而实现，因此占用各种资源几乎可不计，可行！

## Ⅱ. 准备工作

### 2.1 WinDynamicDesktop

WinDynamicDesktop软件可在微软应用商店或通过[Github链接](https://github.com/t1m0thyj/WinDynamicDesktop/releases/tag/v4.7.0)进行下载。

### 2.2 壁纸

所有壁纸均来源于[b站视频·<北京龙江波影视>冬奥会开幕式24节气倒计时段片纯净版](https://www.bilibili.com/video/BV17q4y1b793?spm_id_from=333.1007.top_right_bar_window_history.content.click)，分辨率为3840 * 2160，分为有文字版本和无文字版本各24张，共48张。

[壁纸百度云链接](https://pan.baidu.com/s/1Y3cgUpxcxinpBXWMgHgQBQ)，密码：YYDS

## Ⅲ. 制作

WinDynamicDesktop内置了部分主题，并且可在[官方主题网站](https://windd.info/themes/)（访问较慢，需科学上网）下载免费及付费主题，同时作者还提供了[自制主题的教程](https://github.com/t1m0thyj/WinDynamicDesktop/wiki/Creating-custom-themes)。

### 3.1 教程

阅读教程后，将主题制作步骤总结如下。

**1.图像文件**

主题使用的图像可以是Windows标准的任何文件格式（JPEG、PNG、BMP、GIF 等）。如果想使用一组HEIC图像（macOS 用于动态壁纸的格式），则需要将它们转换为不同的格式，因为Windows本身不支持HEIC。[可以使用此在线图像转换器](https://strukturag.github.io/libheif/)轻松地将它们转换为 JPEG 格式。

要在主题中使用的每个图像都必须具有相同的文件名，但要使用数字来区分它们（例如，`mojave_dynamic_1.jpeg`、`mojave_dynamic_2.jpeg`、 ... `mojave_dynamic_16.jpeg`），并且所有图像文件都在同一个文件夹中。

**2.主题文件**

主题配置文件采用JSON格式，JSON 文件的名称必须是主题的名称，空格替换为下划线，加上`.json`扩展名（例如，`Mojave_Desert.json`）。

官方教程提供了两份模板，分别为4张图片和2张图片。

> 4段主题

```json
{
  "imageFilename": "Big_Sur_*.jpg",
  "imageCredits": "Apple",
  "sunriseImageList": [
    3,
    4
  ],
  "dayImageList": [
    1,
    5,
    6
  ],
  "sunsetImageList": [
    7,
    8
  ],
  "nightImageList": [
    2
  ]
}
```

| 示例日段                         | 显示的图像               |
| -------------------------------- | ------------------------ |
| 日出 (06/15 05:30-07:00)         | #3、4（各 45 分钟）      |
| 当天（06/15 07:00-18:30）        | #1、5、6（各 3.83 小时） |
| 日落 (06/15 18:30-20:00)         | #7、8（每人 45 分钟）    |
| 晚上 (06/15 20:00 - 06/16 05:30) | #2（9.5 小时）         |

> 2段主题

```json
{
  "imageFilename": "Big_Sur_Abstract_*.jpg",
  "imageCredits": "Apple",
  "dayImageList": [
    1
  ],
  "nightImageList": [
    2
  ]
}
```

| 示例日段                         | 显示的图像    |
| -------------------------------- | ------------- |
| 当天（12/01 06:00-17:00）        | #1（11 小时） |
| 晚上 (12/01 17:00 - 12/02 06:00) | #2（13 小时） |

**3.属性**

可以在主题文件中定义以下属性：

- `displayName`- 包含主题的用户友好名称的字符串（可以包含空格）。如果不存在，主题名称将从JSON文件名派生。
- `imageFilename`（必需）- 包含每个壁纸图像文件名的字符串，用星号 ( `*`) 代替图像编号（例如，`mojave_dynamic_*.jpeg`）。
- `imageCredits`（必需）- 包含图像应归属的个人或公司名称的字符串（例如`Apple`，, `Bob Ross`）。
- `dayHighlight`- 图像编号显示在主题预览缩略图的左侧
- `nightHighlight`- 图像编号显示在主题预览缩略图的右侧
- `sunriseImageList`- 列出在整个日出期间显示的图像编号的数字数组（从太阳低于地平线 6 度到高于地平线 6 度）。
- `dayImageList`（必需）- 列出全天（日出和日落之间）显示的图像编号的数字数组。它们将按照您列出它们的顺序显示，每个显示相同的时间长度。如果您列出了 12 张图像，并且日出和日落之间有 12 小时，则每张图像将显示一个小时。
- `sunsetImageList`- 列出在整个日落期间显示的图像编号的数字数组（从太阳高于地平线 6 度到低于地平线 6 度）。
- `nightImageList`（必需）- 列出整晚（日落和日出之间）显示的图像编号的数字数组。行为与`dayImageList`上面描述的相同。

### **3.2 编写JSON文件**

可以看出主题切换的频率主要与日出、日落时间以及持续时间有关，如果我们选择以本地时间日出为时间格式则不能满足每小时切换一张，并且对应相应节气的要求。经过简单计算，我们可以将日出时间设置为早上7时，日落时间设置为晚上7时，持续时间均为2个小时，即可满足要求（此为一种计时方式，其他符合要求的计时方式也能实现）。

编写的JSON格式如下。

```json
{
    "imageFilename": "beijing2022_dynamic_*.png",
    "imageCredits": "BeiJing_2022",
    "sunriseImageList": [
      6,
      7
    ],
    "dayImageList": [
      8,
      9,
      10,
      11,
      12,
      13,
      14,
      15,
      16,
      17
    ],
    "sunsetImageList": [
      18,
      19
    ],
    "nightImageList": [
      20,
      21,
      22,
      23,
      24,
      1,
      2,
      3,
      4,
      5
    ]
  }
```

### 3.3 图片处理

对于图片，我们需要分为有文字版和无文字版，并且同类型图片处于一个文件夹，命名为`beijing2022_dynamic_*.png`，这样我们就可以只编写一个JSON文件应用于两套主题。

### 3.4 应用

最后我们将JSON文件与图片放置于同一文件夹，打开WinDynamicDesktop并设置计时方式为日出时间设置为早上7时，日落时间设置为晚上7时，持续时间均为2个小时。

在选择主题页面，选择从文件中导入。

选择`2022北京冬奥会24节气（有数字）`或`2022北京冬奥会24节气（无数字）`文件夹，选择`BeiJing_2022_num.json`或`BeiJing_2022.json`即可导入主题。

最后选择应用。

![image-20220206204513667](/img/in-post/theme-1.png)

### 3.5 链接

链接：https://pan.baidu.com/s/1FuWWC60cDDNWJK-xADDREA 
提取码：2022

## Ⅳ. 后记

后续在应用有数字版主题时发现部分时间更换壁纸会变为纯黑色背景，经过排查是由于壁纸太大，导致在换壁纸时Windows直接变成了纯黑色背景，最后的解决办法是使用自带的画图调小了分辨率。
