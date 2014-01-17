---
layout: post
title: "NSIS进阶教程(一)"
date: 2013-05-24 17:37
comments: true
categories: NSIS
---

### 自定义界面之无边框窗体移动贴图 ###

## 前言 ##
>
在Windows下，有很多人想做一个完全自己把控的安装程序，想过很多种途径去实现，有人说MFC可以实现，有人说C#可以实现，有人说Delphi可以实现，有人说VB又未尝不可呢。MFC，Delphi，VB，C#都需要自己去实现打包压缩，释放，释放过程中的业务逻辑跟界面功能，是一项比较麻烦的工作，甚至于C#程序需要运行的话，还需要装dotnet Framework的runtime。NSIS制作的安装包可以运行在Win9x下，完全是WinAPI的调用，不需要额外装任何的runtime，安装包双击就能运行，本身封装了很多Win的函数，方便调用与开放接口。功能部分也是实现了基本的安装过程所需的操作，NSIS的很多Editor做到了向导模式的脚本生成，很是方便。这么好的工具能否定制开发呢，答案是肯定的。
<!--more-->
## 本篇主要讲讲以下几点： ##



 + **如何消除普通的NSIS脚本生成的窗体的边框**


 + **如何使得无边框窗体能够移动**

 + **如何给这个窗体贴上一张大小合适的背景图**

**所用到的NSIS插件：**

1. nsdialogs
2. nswindows
3. winproc
4. system

**讲义**：

**首先贴出一个今天教程的完整的例子（附带图片） [猛击这里][1]**

*题外话，本来想用新浪爱问做文件分享平台的，发现上传后一直在审核中……练葵花宝典能力谁也比过性浪呀，用CSDN也不好，还要登录，本人就因为积分太少而不得不去做无聊的工作赢得积分，用于CSDN下载，自从CSDN把我的密码明码保存还被黑客给搞了之后，我不再上此网站。115虽然下载页广告多的一笔，但是后台上传页相当的干净，还不用审核以及无登陆下载，极致方便大家。(115被政府搞了，转性浪爱问)*

### modify:所有源码都转移到Github上面了 ###

***使用时：把插件DLL跟头文件分别放入到你的NSIS本地对应的安装目录中，然后编译源码即可。***

下文都用`%NSIS_Install_DIR%`来替代你本地安装路径

## 去除窗体Border ##

在去除窗体边框之前有一项工作是必须做的，那就是更改默认窗体的大小，因为每个人想做的打包窗体不可能都一样大，更改窗体大小有两种方法，也可以两种方法并用

- **修改NSIS内部的UI**

	NSIS的默认UI放在`%NSIS_Install_DIR%\Contrib\UIs`中，其中常常见到的创建自定义窗体的
`1018`，`1044`都在此路径的`modern.exe`中。我们只要修改`modern.exe`里面的资源文件即可，做过MFC的都知道，VC在创建程序的时候是有Resources的，只要找到一些能更改Resources里面Dialog的工具即可，本文推荐ResHacker 。

	修改的时候宁可大点，也绝不小，因为开发过程中我遇到用`nsWindows`命令扩大窗体的时候，出现不起作用的情况，但是默认窗体比需要的窗体小的时候可以用`nsWindows`命令控制。

	打开`ResHacker`工具拖入`modern.exe`，操作前请备份`modern.exe`，拖动资源窗体或者直接修改你想要的大小。默认的`1044`跟`1018`窗体都在`105`分类下。
![tu][2]
![tu2][3]
- **通过nsWindows命令**

请看代码:

	nsDialogs::Create 1044
	Pop $0
    ${If} $0 == error
        Abort
    ${EndIf}
    SetCtlColors $0 ""  transparent ;背景设成透明
    ${NSW_SetWindowSize} $HWNDPARENT 513 354 ;改变窗体大小
    ${NSW_SetWindowSize} $0 513 354 ;改变Page大小

该脚本添加在自定义窗体的创建Function中，创建的是1044类型窗口，修改命令是两条，分别是对$HWNDPARENT的窗体跟创建的1044page的修改，确保默认的modern.exe的窗口大小比这个要大！
修改好窗体大小后，直接在初始化的Function中直接填入以下代码即可去除边框


	Function onGUIInit
		;消除边框
	    System::Call `user32::SetWindowLong(i$HWNDPARENT,i${GWL_STYLE},0x9480084C)i.R0`
	    ;隐藏一些既有控件
	    GetDlgItem $0 $HWNDPARENT 1034
	    ShowWindow $0 ${SW_HIDE}
	    GetDlgItem $0 $HWNDPARENT 1035
	    ShowWindow $0 ${SW_HIDE}
	    GetDlgItem $0 $HWNDPARENT 1036
	    ShowWindow $0 ${SW_HIDE}
	    GetDlgItem $0 $HWNDPARENT 1037
	    ShowWindow $0 ${SW_HIDE}
	    GetDlgItem $0 $HWNDPARENT 1038
	    ShowWindow $0 ${SW_HIDE}
	    GetDlgItem $0 $HWNDPARENT 1039
	    ShowWindow $0 ${SW_HIDE}
	    GetDlgItem $0 $HWNDPARENT 1256
	    ShowWindow $0 ${SW_HIDE}
	    GetDlgItem $0 $HWNDPARENT 1028
	    ShowWindow $0 ${SW_HIDE}
	FunctionEnd


用`System::Call`命令调用`SetWindowLong`的API函数改变`GWL_STYLE`的样式即可，`System`是NSIS官方插件用于帮助用户调用系统函数，是相当重要的自动安装程序的插件！程序中其余的代码是把创建的`1004`页面上其余的控件给隐藏掉，后面携带的ID都是可以通过`ResHacker`在`105`包中查询到。

## 贴一张大小合适的背景图 ##

贴图需要用到nsDialogs插件的命令：

	;贴背景大图
    ${NSD_CreateBitmap} 0 0 100% 100% ""
    Pop $BGImage
    ${NSD_SetImage} $BGImage $PLUGINSDIR\bg.bmp $ImageHandle
                                                                   
    ${NSD_FreeImage} $ImageHandle
`${NSD_CreateBitmap}`命令创建一个跟窗体一样大小的图片区域，后面的五个参数分别是`x`,`y`,`width`,`height`,`text`，坐标，宽高，文字。紧接着给这张图贴上一张合适的图片`bg.bmp`，贴图片之前需要把这个图片打包到安装程序中，这个是基本的操作，源码包中有，这里就不做说明了。最后还要通过`${NSD_FreeImage}`去释放该图片内存区。

## 无标题移动 ##

做到无标题移动的潜台词是把原本传递给标题栏的`Message`通过你定义的元素回调传递给标题栏，所以只要给你添加的资源加上传递信息的回调函数就可以了。这里是通过`WinProc`这个插件完成的，`WinProc`这个插件在官方的插件库中没有，Google一下就可以查询到，这里的源码包中也有。除了`WinProc`，第三方插件`SkinBtn`也可以帮助实现。

	Function onGUICallback
	  ${If} $MSG = ${WM_LBUTTONDOWN}
	    SendMessage $HWNDPARENT ${WM_NCLBUTTONDOWN} ${HTCAPTION} $0
	  ${EndIf}
	FunctionEnd
以上是回调函数，判断鼠标左键的Down事件，并且传递消息给标题栏。

	GetFunctionAddress $0 onGUICallback
	WndProc::onCallback $BGImage $0 ;处理无边框窗体移动
以上是把当前的$BGImage作为回调主体，当用户左键点击$BGImage的时候，消息就传递给了窗体标题栏，实现了无边框的移动。

**结束语**

看看结果是什么样子的……哦！kugou也被我山寨了一把，有人精益求精，说，你的程序鼠标放在哪里都能移动，人家kugou只能标题栏移动……是的呀，你把图切成几份，分别贴，有的给回调函数，有的不给就实现了消息部分传递的功能。

![over][4]


有一个注意点：贴图的时候注意了！代码的运行是自上而下，如果要贴的图需要在另一张上面的话，需要把代码写在前面。

比如：

	;贴小图
    ${NSD_CreateBitmap} 0 34 100% 100% ""
    Pop $MiddleImage
    ${NSD_SetImage} $MiddleImage $PLUGINSDIR\middle.bmp $ImageHandle
                  
    ;贴背景大图
    ${NSD_CreateBitmap} 0 0 100% 100% ""
    Pop $BGImage
    ${NSD_SetImage} $BGImage $PLUGINSDIR\bg.bmp $ImageHandle


[1]: https://github.com/nicecai/nsissource/tree/master/2 "新浪爱问分享"
[2]: http://m1.img.libdd.com/farm5/57/1AB2FFFC269C4FB01C8802B4D88A2939_479_351.JPEG "exe查看工具图"
[3]: http://m2.img.libdd.com/farm4/7/E783172C1F7B203AE4F838ADFF9C8407_566_435.JPEG "exe查看工具图"
[4]: http://m1.img.libdd.com/farm4/132/5E2C8AA789CBB8EC1CCF5D591DBFB784_517_352.JPEG "over pic"