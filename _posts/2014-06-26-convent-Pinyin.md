---
layout: post
title: 汉字转换为拼音
---

{{ page.title }}
================
<p class="meta">26 June 2014 - tony cap</p>

#汉字转拼音的库主要是：

* pinyin　  　  https://github.com/hotoo/pinyin
* PYMethod      https://github.com/a85816841/PotentialGragonSnail/tree/master/ql/lib/pinying
* POAPinyin     https://github.com/leeeboo/POAPinyin
* PinYin4Objc   https://github.com/kimziv/PinYin4Objc
 
## 实现原理：
* pinyin是把unicode中汉字部分的首字母全部提取到数组,取得时候 拼音数组[汉字的unicode值-unicode中起始汉字值]就直接得到了.
* PYMethod是把unicode转成GBK,然后根据GBK高低位两个值确定对应拼音的位置得到拼音
* POAPinyin是把所有拼音与之对应的汉字组成一个表,到时候往这个表里查询(原生convert方法)改进的quickConvert方法是先得到一个汉字unicode值的上下限,然后转换上面的表成 unicode--拼音 这样的表,查询的时候就是哈希查找,更快,要是这个unicode不连续就会有很大的问题了(这个表里面果然缺了字:"?g?i?k仍?????????????x?z?{????佘????|愣扔?Y楞特????????????????????酿???铽").这个函数还会跳过一些非ascii符号.另一个方法stringConvert修复了非ascii码这个问题.使用的时候最好把上面提到的字加进表里.
 

##比较：

* 大小 pinyin最小了,POAPinyin的声明就快500行了.
* 速度 其实三者差不多,但是不要用POAPinyin原生的那个convert,那个每次都遍历查找很慢.
* 对比 pinyin只能取得汉字对应拼音的首字母,PYMethod原本是应用于股票查询的,它的拼音个数少于POAPinyin.
　　对于这个汉字"嗯",我拼音输入法是"en"打出来的,PYMethod得到的是EN,但是POAPinyin得到的是NG,百度百科也读NG.....

PinYin4Objc 是一个效率很高的汉字转拼音类库，支持简体和繁体中文。
有以下特性：
```
1.效率高，使用数据缓存，第一次初始化以后，拼音数据存入文件缓存和内存缓存，后面转换效率大大提高；
2.支持自定义格式化，拼音大小写等等；
3.拼音数据完整，支持中文简体和繁体，与网络上流行的相关项目比，数据很全，几乎没有出现转换错误的问题。
性能比较：与之前的pinyin，POAPinyin和PYMethod等项目比较，PinYin4Objc的速度是非常快的，差不多为：0.20145秒/1000字 
```
