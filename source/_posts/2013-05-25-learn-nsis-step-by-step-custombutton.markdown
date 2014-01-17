---
layout: post
title: "NSIS进阶教程(二)"
date: 2013-05-25 13:23
comments: true
categories: NSIS
---
### 自定义界面之Button、License窗口实现 ###

## 前言 ##
>
在上一节中我们粗略的处理一下无边框窗体、背景贴图、鼠标移动。这节主要是创建用于响应事件的Button以及能展示软件License的窗口，还能用Button控制软件协议的展示与否。
代码还是延续上一节的。


## 本篇主要讲讲以下几点： ##

+ **如何创建一个自己的按钮**

+ **如何创建一个自己的License窗口**

+ **如何用自己的按钮控制自己的License窗口事件**

**所用到的插件：**

1. nsDialogs

2. SkinBtn

<!--more-->

**讲义**

**首先贴出一个今天教程的完整的例子（附带图片） [猛击这里][1]**

*本文还是在安装程序的首张Welcome中执行，上一节例子中的背景图需要做适当的修改，因为原本需要创建Button的地方背后有图，适当用PS去抹去就好。本文的前期工具就是PS了一部分元素。加入了几张Button的背景图。*

## 创建一个属于自己Design的Button ##

	;下一步
    ${NSD_CreateButton} 319 303 88 25 ""
        Pop $Btn_Next
        StrCpy $1 $Btn_Next
        Call SkinBtn_Next
        GetFunctionAddress $3 onClickNext
    SkinBtn::onClick $1 $3
原本`nsDialogs`插件就可以用`${NSD_CreateButton}`命令创建一个`Button`按钮出来，可惜的是此`Button`按钮默认的有点丑。

下面就要说到如何改观`Button`的样式，再接触之前我一直想找有没有用一张图片来替代`Button`的区域，并且给`Image`附上事件之类的想法。等到刘若英的后来，我发现有一个`SkinBtn`的插件可以完成此类工作。

`SkinBtn`是我在梦想吧中下载的一个第三方插件，国人开发，支持`Button`的五种状态：`Normal`，`Hover`，`Click`，`Disabled`，`Focus`，正常、鼠标悬浮、单击时、不可用时、获得焦点时。这五种状态已经完全够用了。

所要注意的就是`SkinBtn`的五种状态图需要竖行排列，宽高没有限制，但是高度像素最好的5的倍数（我猜是为了整除，方便取图）下面贴一张示例`Button`的贴图：

![Button贴图][2]

SkinBtn的用法是这样的：

在`.onInit`的初始化图片资源，并且要用`SkinBtn`插件的方法初始化一下：

 

	File `/ONAME=$PLUGINSDIR\btn_next.bmp` `images\btn_next.bmp`
	SkinBtn::Init "$PLUGINSDIR\btn_next.bmp"
紧接着`Call`以下方法，给当前的`Button`附上定义的图形：

	Function SkinBtn_Next
	  SkinBtn::Set /IMGID=$PLUGINSDIR\btn_next.bmp $1
	FunctionEnd
现在窗体中的按钮已经被处理成自己想要的效果了，这时候衣服算是穿起来了，我们再试一试他的功能怎么样，我们定义一个`onClickNext`的`function`，然后调用`SkinBtn`插件的`onClick`方法进行调用：

	GetFunctionAddress $3 onClickNext
	    SkinBtn::onClick $1 $3
	;下一步按钮事件
	Function onClickNext
	  MessageBox MB_OK "下一步"
	  Abort
	FunctionEnd
单击事件里面只做了弹出消息框这么一个临时的操作，后面的教程中会说明怎么弹到另一张page中，今天这里不谈。

这样一个自己的贴图的Button，从外观到功能都已经完成了。

## 创建显示License内容的控件 ##

关于License的框如何显示的问题，NSIS的默认窗体有一个专门的协议`Page`，不过在我们的自定义`Page`中就不说那个了，在我们的Welcome页面中加入一个协议显示的窗口，这时候需要准备一个协议按钮的贴图，协议窗口。

![协议贴图][3]

![协议贴图][4]

协议的按钮样子看上去是超链接，我们可以通过Button贴图去实现，不要一味认死理去创建超链接的命令。因为有两种状态，所以有两份贴图。

图片预载入跟贴图的步骤就不说了，这里初始化该Button的时候是默认的箭头向上的图，我们只要额外的创建一个变量来控制点击的状态，并且在按钮点击事件中重新给Button贴上第二张箭头向上的图就可以实现按钮的贴图跟功能的统一：

	Function SkinBtn_Agreement1
	  SkinBtn::Set /IMGID=$PLUGINSDIR\btn_agreement1.bmp $1
	FunctionEnd
                                                                                                  
	Function SkinBtn_Agreement2
	  SkinBtn::Set /IMGID=$PLUGINSDIR\btn_agreement2.bmp $1
	FunctionEnd
                                                                                                  
	;协议按钮事件
	Function onClickAgreement
	    ${IF} $Bool_License == 1
	        IntOp $Bool_License $Bool_License - 1
	        StrCpy $1 $Btn_Agreement
	        Call SkinBtn_Agreement1
	    ${ELSE}
	        IntOp $Bool_License $Bool_License + 1
	        StrCpy $1 $Btn_Agreement
	        Call SkinBtn_Agreement2
	    ${EndIf}
	FunctionEnd
以上代码中，`onClickAgreement`方法是`Button`的点击响应事件，变量`Bool_License`是控制显示那张`Button`背景图的控制单元，它的初始值是`0`，`IF`判断中写的是当前状态下需要处理的事件。`IntOp`命令是执行加减运算。

这样，一个根据点击来更改背景图片的按钮就做好了。

## 用自定义的按钮去控制`License`窗口的显示 ##

在做`License`之前首先要准备一份`License`的文件，NSIS命令中有读取文件，显示文件内容的命令，该文件可以是Txt，也可以是带有格式的RTF，这里示例中选用RTF文件。在源码包的RTF协议用的KuGou的。而用于显示RTF的控件则是`nsDialogs`插件提供的`RichEdit20A`，还要注意引入头文件`LoadRTF.nsh`

	;读取RTF的文本框
    nsDialogs::CreateControl "RichEdit20A" \
    ${ES_READONLY}|${WS_VISIBLE}|${WS_CHILD}|${WS_TABSTOP}|${WS_VSCROLL}|${ES_MULTILINE}|${ES_WANTRETURN} \
    ${WS_EX_STATICEDGE} \
    5 44 500 203 ''
    Pop $Txt_License
    ${LoadRTF} '$PLUGINSDIR\license.rtf' $Txt_License
    ShowWindow $Txt_License ${SW_HIDE}
以上代码就是创建读取RTF文件的多行文本框，并且设定了一些常用的属性，具体的样式属性可以通过查询`nsDialogs`插件的官方文件可以得到，我们让该控件默认是不显示的状态。

这样一个，协议的显示控件就创建完毕了。

创建好了协议控件`$Txt_License`，那么接着就是用刚才创建的`Button`控制该窗口的显示与否，修改刚才的代码：

	;协议按钮事件
	Function onClickAgreement
	    ${IF} $Bool_License == 1
	        ShowWindow $MiddleImage ${SW_SHOW}
	        ShowWindow $Txt_License ${SW_HIDE}
	        IntOp $Bool_License $Bool_License - 1
	        StrCpy $1 $Btn_Agreement
	        Call SkinBtn_Agreement1
	    ${ELSE}
	        ShowWindow $MiddleImage ${SW_HIDE}
	        ShowWindow $Txt_License ${SW_SHOW}
	        IntOp $Bool_License $Bool_License + 1
	        StrCpy $1 $Btn_Agreement
	        Call SkinBtn_Agreement2
	    ${EndIf}
	FunctionEnd

在事件当中，加入隐藏中间背景图跟显示`License`窗口的代码，这样一个修改`Button`贴图跟控制协议窗口显示与否的关联就做好了。

**结束语**

![整体图][5]
![整体图][6]





图片还需修整修整。

在此文中，核心的部分是如何创建读取RTF的协议窗口还有Button控制协议窗口的显示的部分，这部分所用到的代码逻辑是每个初级Coder都可以写出来的，本文也就是抛砖引玉。

[1]: https://github.com/nicecai/nsissource/tree/master/2 "例子下载"
[2]: http://m1.img.libdd.com/farm5/247/02C129DE5A48EEB00210DA62142D77F7_88_125.jpg "Button贴图"
[3]: http://m3.img.libdd.com/farm5/177/FC7EB1FC677FFC6327EECA787CE628B1_95_75.jpg "协议贴图"
[4]: http://m2.img.libdd.com/farm4/18/9D7F4E4676411D39EEEEA26CFCA46312_95_75.jpg "协议贴图"

[5]: http://m3.img.libdd.com/farm5/228/0A8488AC09A0817F2629BC3AAE6C29E4_515_352.JPEG "整体图"
[6]: http://m1.img.libdd.com/farm5/40/EF24BBA77B190D6A60852983C5A46828_515_352.JPEG "整体图"