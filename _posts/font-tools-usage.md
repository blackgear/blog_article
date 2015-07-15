title: FontTools安装与使用简明指南
date: 2015-06-26 23:30:00
categories: font
tags: [font, ttx]
---

## 相关说明

FontTools是一套以ttx为核心的工具集，用于处理与字体编辑有关的各种问题，程序用Python编写完成，代码开源，具有良好的跨平台性。FontTools由以下4个程序组成：

- ttx 可将字体文件与xml文件进行双向转换
- pyftmerge 可将数个字体文件合并成为一个字体文件
- pyftsubset 可产生一个由字体的指定字符组成的子集
- pyftinspect 可显示字体文件的二进制组成信息

## 安装FontTools

FontTools原本是托管在[Sourceforge](http://sourceforge.net/projects/fonttools/)上的项目，由于长期不再更新，于是Behdad在[Github](https://github.com/behdad/fonttools)上建立了fork，并继续进行开发。建议选择从Github下载FontTools。

Windows用户需要首先安装Python 2.7，py2exe，从[Releases页面](https://github.com/behdad/fonttools/releases)下载FontTools，解压缩，随后在Setup.py的倒数第二行，`**classifiers`的上方加入：

    options = {'py2exe': {'bundle_files': 2, 'compressed': True}},
    console = [{'script':'Tools/ttx'}],

随后运行：

    $ setup.py py2exe

即可在dist目录下找到ttx.exe。

Windows用户也可以从[这里](https://darknode.in/dl/ttx.zip)直接下载我编译好的ttx.exe使用。

Mac、Linux用户从[Releases页面](https://github.com/behdad/fonttools/releases)下载FontTools，解压缩，随后执行：

    $ python setup.py install

即可完成安装。

Mac用户也可以选择使用Homebrew直接完成安装：

    $ brew install fonttools

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

## TTX使用说明

ttx是FontTools的核心工具，用于将字体转换为xml文件，或者将xml文件转换回字体。ttx所产生的xml文件的后缀名是ttx，可以用各种文档编辑器打开进行编辑。

最基本的用法是这样的：

    $ ttx font.ttf
    $ ttx font.ttx

这样就能将字体全部的表转换为ttx文件。或是将ttx文件转换回字体文件了。

ttx可以加入一些参数使用，常见的参数包括以下一些：

    $ ttx -l font.ttf

`-l`参数用于显示字体包含哪些表。

    $ ttx -t name font.ttf

`-t`参数用于将字体中的name表转换为ttx文件。

`-b`参数用于指定合并时不重新计算字框参数而是直接使用原来的参数。
