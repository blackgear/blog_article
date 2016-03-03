title: CMap表相关修改技术简要指南
date: 2015-07-10 02:00:00
categories: font
tags: [font, ttx, cmap]
description: 本文介绍了如何使用ttx修改字体的cmap表，以增强字体在不同操作系统中的兼容性。
---

## 基本信息

CMap表是用于将字符编码映射到字形索引的表，字体渲染过程中，首先会获得要渲染的字符的字符Unicode编码，随后根据CMap表找到其对应的字形的索引值，再根据索引值读取对应字形的数据并进行渲染。其具体信息可以参考[Apple文档](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6cmap.html)和[Windows文档](https://www.microsoft.com/typography/otspec/cmap.htm)。

## CMap表细节解说

CMap表可以通过ttx的命令导出：

    ttx -t cmap Font.ttf

用ttx导出的CMap表通常是这个样子的：

    <cmap>
        <tableVersion version="0"/>
        <cmap_format_4 platformID="0" platEncID="3" language="0">
            <map code="0x20" name="space"/><!-- SPACE -->
            <map code="0x21" name="exclam"/><!-- EXCLAMATION MARK -->
            <map code="0x22" name="quotedbl"/><!-- QUOTATION MARK -->
            <map code="0x23" name="numbersign"/><!-- NUMBER SIGN -->
            <map code="0x24" name="dollar"/><!-- DOLLAR SIGN -->
            <map code="0x25" name="percent"/><!-- PERCENT SIGN -->
            <map code="0x26" name="ampersand"/><!-- AMPERSAND -->
            <map code="0x27" name="quotesingle"/><!-- APOSTROPHE -->
    …
        <cmap_format_4 platformID="3" platEncID="1" language="0">
            <map code="0x20" name="space"/><!-- SPACE -->
            <map code="0x21" name="exclam"/><!-- EXCLAMATION MARK -->
            <map code="0x22" name="quotedbl"/><!-- QUOTATION MARK -->
            <map code="0x23" name="numbersign"/><!-- NUMBER SIGN -->
            <map code="0x24" name="dollar"/><!-- DOLLAR SIGN -->
            <map code="0x25" name="percent"/><!-- PERCENT SIGN -->
            <map code="0x26" name="ampersand"/><!-- AMPERSAND -->
            <map code="0x27" name="quotesingle"/><!-- APOSTROPHE -->

CMap表中通常包括数个子表，比如上面的例子中就有两个子表，分别是：

    <cmap_format_4 platformID="0" platEncID="3" language="0">
    <cmap_format_4 platformID="3" platEncID="1" language="0">

format platformID platEncID这三组值指定了CMap表数据的编码方式与顺序，常见的字体通常具有多个子表。幸运的是，它们通常只有4个常见的组合：

* cmapFormat=4 platformID=0 platEncID=3: Unicode UCS-2
* cmapFormat=4 platformID=3 platEncID=1: Windows UCS-2
* cmapFormat=12 platformID= 0 platEncID=4: Unicode UCS-4
* cmapFormat=12 platformID= 3 platEncID=10: Windows UCS-4

首先，format 4只支持65536个字符，format 12是format 4的超集，支持2147483648个字符。英文字体通常只使用format 4，CJK字体则常常会用到format 12。在使用format 12格式时仍然需要保留一个对应的format 4格式的子表，否则Windows下将出现字体兼容性问题。

platformID 0表示Unicode平台，而platformID 3表示Windows平台，大多数字体渲染引擎或是程序会优先读取Unicode平台的CMap，但是Windows至今仍不支持Unicode平台，需要保留Windows平台的子表以确保兼容性。

platEncID与platformID有关，表示在对应平台下的具体编码方式。其具体取值如下：

* platformID=0 platEncID=0: Default semantics
* platformID=0 platEncID=1: Version 1.1 semantics
* platformID=0 platEncID=2: ISO 10646 1993 semantics (deprecated)
* platformID=0 platEncID=3: Unicode 2.0 or later semantics (BMP only)
* platformID=0 platEncID=4: Unicode 2.0 or later semantics (non-BMP characters allowed)
* platformID=0 platEncID=5: Unicode Variation Sequences
* platformID=0 platEncID=6: Full Unicode coverage (used with type 13.0 cmaps by OpenType)

* platformID=3 platEncID=0: Symbol
* platformID=3 platEncID=1: Unicode BMP (UCS-2)
* platformID=3 platEncID=2: ShiftJIS
* platformID=3 platEncID=3: PRC
* platformID=3 platEncID=4: Big5
* platformID=3 platEncID=5: Wansung
* platformID=3 platEncID=6: Johab
* platformID=3 platEncID=7: Reserved
* platformID=3 platEncID=8: Reserved
* platformID=3 platEncID=9: Reserved
* platformID=3 platEncID=10: Unicode UCS-4

这些组合当中，cmapFormat=4 platformID=0 platEncID=3与cmapFormat=4 platformID=3 platEncID=1等价，cmapFormat=12 platformID=0 platEncID=4与cmapFormat=12 platformID=3 platEncID=10等价，表示相同的编码方式与顺序。

子表内容由许多条记录构成，比如：

    <map code="0x25" name="percent"/><!-- PERCENT SIGN -->

表示Unicode编码为0x25的字符被对应到字体文件中percent这个字形。

## PingFang的修改

PingFang是Apple公司在OS X 10.11中新加入的字体，在最初的DP1版本中，只需对ttc文件进行解包即可在Windows下正常使用，而DP2之后的版本却不能这样，其根本原因是DP1版本的PingFang有以下4个CMap子表：

    <cmap_format_4 platformID="0" platEncID="3" language="0">
    <cmap_format_12 platformID="0" platEncID="4" format="12" reserved="0" length="185584" language="0" nGroups="15464">
    <cmap_format_4 platformID="3" platEncID="1" language="0">
    <cmap_format_12 platformID="3" platEncID="10" format="12" reserved="0" length="185584" language="0" nGroups="15464">

而DP2之后的版本却变成了2个CMap子表：

    <cmap_format_4 platformID="0" platEncID="3" language="0">
    <cmap_format_12 platformID="0" platEncID="4" format="12" reserved="0" length="189520" language="0" nGroups="15792">

 由于缺乏platformID为3的子表，Windows将其视为了无效的字体文件。

根据前面的介绍，将DP2之后的字体文件修改为兼容Windows的字体文件的方法就是加入对应的表了，不过我们有个偷懒的办法：将cmapFormat=4 platformID=0 platEncID=3直接改成cmapFormat=4 platformID=3 platEncID=1，将cmapFormat=12 platformID=0 platEncID=4直接改为cmapFormat=12 platformID=3 platEncID=10，由于这两组对应的编码方式与顺序完全相同，所以我们并不需要修改后续的子表内容就能使这个字体在Win下可用。

先使用otc2otf将PingFang.ttc解包，随后在OS X下执行：

    ttx -t cmap PingFang-SC-Regular.otf
    sed -i '' 's/platformID="0" platEncID="3"/platformID="3" platEncID="1"/g' PingFang-SC-Regular.ttx
    sed -i '' 's/platformID="0" platEncID="4"/platformID="3" platEncID="10"/g' PingFang-SC-Regular.ttx
    ttx -b -m PingFang-SC-Regular.otf PingFang-SC-Regular.ttx

即可使这些otf文件在Windows下可用。

## Hiragino Sans的修改

Hiragino Sans也是OS X 10.11中新引入的字体。它的前身是Hiragino Kaku Gothic，在加入多个新字重，补全为从W0至W9的庞大家族后，更名为Hiragino Sans。这个字体的CMap表有3个子表：

    <cmap_format_14 platformID="0" platEncID="5" format="14" length="23120" numVarSelectorRecords="6">
    <cmap_format_2 platformID="1" platEncID="1" language="0">
    <cmap_format_12 platformID="3" platEncID="10" format="12" reserved="0" length="128992" language="0" nGroups="10748">

由于CMap表中缺乏format 4的子表，这个字体无法被Windows识别，其修改方式比较麻烦，需要手动加入一个新的子表并写入对应内容才行。

我们首先关注已有的format 12的子表：

    <cmap_format_12 platformID="3" platEncID="10" format="12" reserved="0" length="128992" language="0" nGroups="10748">
        <map code="0x0" name="cid00001"/><!-- ???? -->
        <map code="0x1" name="cid00001"/><!-- ???? -->
        <map code="0x2" name="cid00001"/><!-- ???? -->
        <map code="0x3" name="cid00001"/><!-- ???? -->
        <map code="0x4" name="cid00001"/><!-- ???? -->
    …
        <map code="0xffe8" name="cid00323"/><!-- HALFWIDTH FORMS LIGHT VERTICAL -->
        <map code="0x1f100" name="cid08061"/><!-- ???? -->
    …
        <map code="0x2f920" name="cid07839"/><!-- ???? -->

之前提到过，cmapFormat=4 platformID=3 platEncID=1这个子表其实就是cmapFormat=12 platformID=3 platEncID=10这个子表的子集，确切的说是其0x0000至0xffff的部分，那么加入新的子表就很简单了。

把map code从0x0000一直到0xffff的段落全部复制下来，粘贴到新加入的format 4的子表之后：

    <cmap_format_4 platformID="0" platEncID="3" language="0">
            <map code="0x0" name="cid00001"/><!-- ???? -->
        <map code="0x1" name="cid00001"/><!-- ???? -->
        <map code="0x2" name="cid00001"/><!-- ???? -->
        <map code="0x3" name="cid00001"/><!-- ???? -->
        <map code="0x4" name="cid00001"/><!-- ???? -->
    …
        <map code="0xffe8" name="cid00323"/><!-- HALFWIDTH FORMS LIGHT VERTICAL -->

最后将修改后的ttx文件合并回源文件，到此为止，其兼容Windows的版本就修改完成了。
