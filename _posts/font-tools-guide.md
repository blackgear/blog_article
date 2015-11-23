title: FontTools安装与使用简明指南
date: 2015-06-26 23:30:00
categories: font
tags: [fonttools, ttx, modify]
description: 本文介绍了在不同系统上安装ttx系列软件，并使用ttx进行字体修改的初步思路。
---

## 相关说明

FontTools是一套以ttx为核心的工具集，用于处理与字体编辑有关的各种问题，程序用Python编写完成，代码开源，具有良好的跨平台性。FontTools由以下4个程序组成：

- ttx 可将字体文件与xml文件进行双向转换
- pyftmerge 可将数个字体文件合并成为一个字体文件
- pyftsubset 可产生一个由字体的指定字符组成的子集
- pyftinspect 可显示字体文件的二进制组成信息

FontTools原本是托管在[Sourceforge](http://sourceforge.net/projects/fonttools/)上的项目，由于原项目长期停滞，Behdad在[Github](https://github.com/behdad/fonttools)上建立了fork，并继续进行开发。由于FontTools基于Python写成，在安装FontTools之前需要首先安装Python。

## 安装Python

Linux与OS X用户系统中已预先安装了Python，可以跳过此步骤。

Windows用户首先需要从[Downloads](https://www.python.org/downloads/)界面下载Python 2.7并安装，在Customize Python这一安装步骤中点击最后的`Add python.exe to Path`这一选项，并选择`Will be installed on local hard drive`。

## 安装FontTools

OS X用户推荐使用Homebrew直接完成安装：

    $ brew install fonttools

Windows、Linux、OS X用户也可从[Releases页面](https://github.com/behdad/fonttools/releases)下载FontTools最新版本，解压缩，随后执行命令进行安装：

    $ python setup.py install

## 打包为可独立运行的exe文件

此步骤将生成一个可独立运行的ttx.exe，pyftmerge.exe，pyftsubset.exe。

Windows用户需要首先安装Python 2.7，py2exe，从[Releases页面](https://github.com/behdad/fonttools/releases)下载FontTools，解压缩，随后在Setup.py的倒数第二行，`**classifiers`的上方加入：

    options = {'py2exe': {'bundle_files': 2, 'compressed': True}},
    console = [{'script':'Tools/ttx'}, {'script':'Tools/pyftmerge'}, {'script':'Tools/pyftsubset'}],

随后运行：

    $ python setup.py py2exe

即可在dist目录下找到生成的文件，其中ttx.exe，pyftmerge.exe，pyftsubset.exe，python27.dll，library.zip这五个文件需要保留，其他文件可以删除。

Windows用户也可以从[这里](https://darknode.in/dl/FontTools.7z)直接下载我编译好的程序包。

## 字体基本知识

一个字体由数个表（table）构成，字体的信息储存在表中。一个最基本的字体文件一定会包含以下的表：

| Tag  | Name                              |
| ---- | --------------------------------- |
| cmap | Character to glyph mapping        |
| head | Font header                       |
| hhea | Horizontal header                 |
| hmtx | Horizontal metrics                |
| maxp | Maximum profile                   |
| name | Naming table                      |
| OS/2 | OS/2 and Windows specific metrics |
| post | PostScript information            |

使用TrueType曲线绘制的字体会包含如下的表：

| Tag  | Name                                          |
| ---- | --------------------------------------------- |
| cvt  | Control Value Table                           |
| fpgm | Font program                                  |
| glyf | Glyph data                                    |
| loca | Index to location                             |
| prep | CVT Program                                   |
| gasp | Grid-fitting/Scan-conversion (optional table) |

使用PostScript曲线绘制的字体会包含如下的表：

| Tag  | Name                                          |
| ---- | --------------------------------------------- |
| CFF  | PostScript font program (compact font format) |
| VORG | Vertical Origin (optional table)              |

使用SVG曲线绘制的字体会包含如下的表：

| Tag | Name                                     |
| --- | ---------------------------------------- |
| SVG | The SVG (Scalable Vector Graphics) table |

使用Bitmap图形构成的字体会包含如下的表：

| Tag  | Name                          |
| ---- | ----------------------------- |
| EBDT | Embedded bitmap data          |
| EBLC | Embedded bitmap location data |
| EBSC | Embedded bitmap scaling data  |
| CBDT | Color bitmap data             |
| CBLC | Color bitmap location data    |

包含高级书法特性的字体会包含如下的表：

| Tag  | Name                    |
| ---- | ----------------------- |
| BASE | Baseline data           |
| GDEF | Glyph definition data   |
| GPOS | Glyph positioning data  |
| GSUB | Glyph substitution data |
| JSTF | Justification data      |
| MATH | Math layout data        |

包含其他特性的字体会包含如下的表：

| Tag  | Name                      |
| ---- | ------------------------- |
| DSIG | Digital signature         |
| hdmx | Horizontal device metrics |
| kern | Kerning                   |
| LTSH | Linear threshold data     |
| PCLT | PCL 5 data                |
| VDMX | Vertical device metrics   |
| vhea | Vertical Metrics header   |
| vmtx | Vertical Metrics          |
| COLR | Color table               |
| CPAL | Color palette table       |

对字体的修改基本上是围绕着最上方的基本表进行的，如果要修改字符形态等才需要用到之后的表。

## ttx使用说明

ttx是FontTools的核心工具，用于将字体转换为xml文件，或者将xml文件转换回字体。ttx所产生的xml文件的后缀名是ttx，可以用各种文档编辑器打开进行编辑。

ttx的基本用法是这样的：

    $ ttx font.ttf
    $ ttx font.ttx

前者将字体全部的表转换为ttx文件。后者则将ttx文件转换回字体文件。

ttx可以加入一些参数使用，常见的参数包括以下一些：

    $ ttx -l font.ttf

`-l`参数用于显示字体包含哪些表。

    $ ttx -t name font.ttf

`-t`参数用于将字体中的name表转换为ttx文件。

    $ ttx -b -m font.ttf font.ttx

`-b`参数用于指定合并时不重新计算字框参数而是直接使用原来的参数。
`-m`参数用于将只包含部分表的ttx文件合并到原有的字体文件中。

## pyftmerge使用说明

目前pyftmerge的开发还停留在早期阶段，使用如下命令即可合并字体：

    $ pyftmerge font1.otf font2.otf

其配置选项与具体实现皆不明确，使用需谨慎。

## pyftsubset使用说明

pyftsubset可提取一个字体的部分字符，产生一个只由它们组成的新字体。通过这一子集化技术，可有效缩小字体文件的体积，便于网络传输。

pyftsubset的基本用法是这样的：

    $ pyftsubset font.otf --text="汉字"

`--text`选项用于指定需要保留的字符

`--text-file`选项用于指定一个包含需要保留的字符的txt文档

`--output-file`选项用于指定输出文件的保存位置
