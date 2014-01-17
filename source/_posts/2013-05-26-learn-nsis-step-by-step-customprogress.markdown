---
layout: post
title: "NSIS进阶教程(四)"
date: 2013-05-26 11:33
comments: true
categories: NSIS
---

### 自定义目录选择，自定义进度条，自定义图片切换效果 ###

## 前言 ##
>
上一节中我们已经处理了有关CheckBox自定义贴图的部分，但是目录选择的部分还没有加上，这节，我们先处理一下目录的选择部分，选择完路径之后就剩下安装了，于是进度条的创建的显得很有必要，但是系统的进度条创建简单，如何改变进度条的背景色跟进度色呢，这节我们也处理掉。有关图片切换的效果的插件也有很多，但是大部分都是基于默认安装窗体进行的，如何在一个完全自定义的页面上面创建一个图片切换效果呢，这节也可以得到答案。

## 本篇主要讲讲以下几点： ##

+ **创建目录选择按钮与文本框**

+ **创建自定义的进度条**

+ **创建图片切换效果**

**所用到的插件【新增】：**

1. WebCtrl

2. SkinProgress

3. 	BgWorker
<!--more-->
**讲义**

**首先贴出今天教程的完整的例子【已测试】【附带图片】 [猛击这里][1]**

*这次的教程基本是安装界面的最后的阶段的处理了，主要是创建一个目录选择控件，用于确定安装程序安装的路径，在安装过程中等待时间也是比较长的，遇到打广告的好时机不要错过，切换多图是个不错的选择；在安装过程中，释放文件是一个主要动作。如果不是自定义页面，该动作是在系统的安装页面的Section里面完成的，如果不做多线程处理，该动作会阻塞主线程也就是主界面的消息传递，主界面没有了消息传递也就不会响应拖动、点击关闭等等操作，这个是本节主要说明的地方。最后说一下图片切换的实现，以我目前尝试的，效果最好的还算是直接放一张网页最实在，通过网页的js实现切换效果。*

## 目录选择框 ##

目录选择框包括一个文本框跟一个Button按钮，做过Web的人都知道，html中有一个file的控件与之相吻合，在form中则是两个不同的component，文本框很好创建，按钮的单击事件是一个重点，我们看看代码：

	;更改目录控件创建
	    ${NSD_CreateDirRequest} 26 79 358 25 "$INSTDIR"
	    Pop $Txt_Browser
	    ${NSD_OnChange} $Txt_Browser OnChange_DirRequest
	                                                                                                                                                                        
	    ${NSD_CreateBrowseButton} 400 79 88 25 ""
	    Pop $Btn_Browser
	    StrCpy $1 $Btn_Browser
	    Call SkinBtn_Browser
	    GetFunctionAddress $3 OnClick_BrowseButton
	      SkinBtn::onClick $1 $3
这里创建了一个`$Txt_Browser`的文本框，一个`$Btn_Browser`的按钮，其中按钮的单击事件是 `OnClick_BrowserButton`，看看按钮的事件代码:

	Function OnClick_BrowseButton
	  Pop $0
	                                                                                                                                                                 
	  Push $INSTDIR 
	  Call GetParent
	  Pop $R0
	                                                                                                                                                                 
	  Push $INSTDIR
	  Push "\"
	  Call GetLastPart
	  Pop $R1
	                                                                                                                                                                 
	  nsDialogs::SelectFolderDialog "请选择 $R0 安装的文件夹:" "$R0"
	  Pop $0
	  ${If} $0 == "error" 
	    Return
	  ${EndIf}
	  ${If} $0 != ""
	    StrCpy $INSTDIR "$0\$R1"
	    system::Call `user32::SetWindowText(i $Txt_Browser, t "$INSTDIR")`
	  ${EndIf}
	FunctionEnd
                                                                                                                                                            
	;得到选中目录用于拼接安装程序名称
	Function GetParent
	  Exch $R0
	  Push $R1
	  Push $R2
	  Push $R3
	  StrCpy $R1 0
	  StrLen $R2 $R0
	  loop:
	    IntOp $R1 $R1 + 1
	    IntCmp $R1 $R2 get 0 get
	    StrCpy $R3 $R0 1 -$R1
	    StrCmp $R3 "\" get
	    Goto loop
	  get:
	    StrCpy $R0 $R0 -$R1
	    Pop $R3
	    Pop $R2
	    Pop $R1
	    Exch $R0
	FunctionEnd
	                                                                                                                                                                 
	                                                                                                                                                           
	;截取选中目录
	Function GetLastPart
	  Exch $0 ; chop char
	  Exch
	  Exch $1 
	  Push $2
	  Push $3
	  StrCpy $2 0
	  loop:
	    IntOp $2 $2 - 1
	    StrCpy $3 $1 1 $2
	    StrCmp $3 "" 0 +3
	      StrCpy $0 ""
	      Goto exit2
	    StrCmp $3 $0 exit1
	    Goto loop
	  exit1:
	    IntOp $2 $2 + 1
	    StrCpy $0 $1 "" $2
	  exit2:
	    Pop $3
	    Pop $2
	    Pop $1
	    Exch $0 
	FunctionEnd
单击按钮的时候用`nsDialogs`创建一个`SelectFolderDialog`的对话框，选择需要安装的路径。`GetParent`方法主要是要取得当前选中的路径，`GetLastPart`主要是保留当前的安装程序最底层的目录名称，最终两个部分合起来作为整体的安装路径赋值给`$INSTDIR`。具体的效果可以运行源码查看。             

## 创建自定义进度条 ##

`nsDialogs`有自带的进度条，该进度条的颜色是那种系统的颜色，如果遇到自己设计的，就会出现问题，于是用`SkinProgress`就可以给进度条赋上两张图片，一张是底图，一张是进度图，虽然不是非常的完美，比如圆角，比如透明，比如阴影，比如……但是已经是跟NSIS快捷脚本配合的很好的了。

	${NSD_CreateProgressBar} 24 265 474 7 ""
	    Pop $PB_ProgressBar
	    SkinProgress::Set $PB_ProgressBar "$PLUGINSDIR\loading2.bmp" "$PLUGINSDIR\loading1.bmp"
用法非常简单，用`nsDialogs`创建一个`ProgressBar`，然后用`SkinProgress`去`set`一下，后面跟上两张图片。`ProgressBar`的图片是有讲究的，具体的可以看源码的image文件夹中两张图片的切图，基本要使用什么效果，自己做两张图就可以了。                           

## 创建图片切换 ##

图片的切换有很多种，`gif`、`flash`、`js`、甚至自己用c++做插件实现，这里提供一种很方面，但是又不失功能强大的方式。网页的形式！你很容易能自己写一段js实现多张图片的切换，这里唯一要解决的就是如何在`Form`上创建一个浏览器用于加载自己的本地`html`页面。

	System::Call `*(i,i,i,i)i(1,34,518,200).R0`
	    System::Call `user32::MapDialogRect(i$HWNDPARENT,iR0)`
	    System::Call `*$R0(i.s,i.s,i.s,i.s)`
	    System::Free $R0
	    FindWindow $R0 "#32770" "" $HWNDPARENT
	    System::Call `user32::CreateWindowEx(i,t"STATIC",in,i${DEFAULT_STYLES}|${SS_BLACKRECT},i1,i34,i518,i200,iR0,i1100,in,in)i.R0`
	    StrCpy $WebImg $R0
	    WebCtrl::ShowWebInCtrl $WebImg "$PLUGINSDIR/index.htm"
首先在界面上创建一个`STATIC`的对象(对话框)，通过`MapDialogRect`来定位该对话框的位置(`1,34`)，大小(`518,200`)，然后通过`WebCtrl`控件来把自己的网页`index.htm`加载进去，`WebCtrl`插件实现的就是调用本地浏览器。这里定位浏览器的位置以及大小是难点，还需多多熟悉才是。

网页里面的代码我就不讲解了，主要是图片js切换，你也可以更换js代码，网络中很多效果都有。

如果发现界面上的图片切换画面跟外边框有距离，就去查看网页里面的`css`，是否把`margin`跟`padding`都设成了0，这样就没有间隙了，看上去跟贴在form上面的一样。：）                                     

## 多线程安装 ##

界面都画好了，现在说到安装文件的时候释放了。源码中我是通过`sleep`来模拟的，具体的释放跟这个差不多。首先创建的是一个只运行一次的定时器`Timer`：
	
	GetFunctionAddress $0 NSD_TimerFun
	    nsDialogs::CreateTimer $0 1
其中`nsDialogs::CreateTimer`后面的参数1代表的是1毫秒，正常情况下就是1毫秒执行一次`NSD_TimerFun`介个方法。

	Function NSD_TimerFun
	    GetFunctionAddress $0 NSD_TimerFun
	    nsDialogs::KillTimer $0
	    !if 1   ;是否在后台运行,1有效
	        GetFunctionAddress $0 InstallationMainFun
	        BgWorker::CallAndWait
	    !else
	        Call InstallationMainFun
	    !endif
	FunctionEnd
本身定时器就是一个异步的功能，如果定时器是同步的，那就没有意义了对不，那么既然是异步的，为什么在`NSD_TimerFun`里面做`Sleep`操作的时候，主界面会卡死不响应呢。这个就要看看`nsDialogs`插件如何实现这个定时器的了，我也没有详查，应该不是创建子线程`Thread`的方式。

这里如果抛弃`Timer`，直接使用`BgWorker`的话，也可以，但是会有一个缺陷：当你需要同时运行两个方法的时候，只有通过创建两个`Timer`来实现，并且在`Timer`调用的方法里面采用`BgWorker`来实现子线程操作。

看看上面代码在`Timer`执行方法里面，第一步是停止`Timer`：因为我们只用了他的异步的特点，并没有想做定时器。然后使用`BgWorker`插件。

使用`BgWorker`插件非常简单，只需用`$0`接收方法地址，然后调用`BgWorker::CallAndWait`。顾明思意，`BgWorker`调用过程是同步的，而且会`Wait`到`InstallationMainFun`方法执行结束，如果`BgWorker::CallAndWait`调用下面还有代码的话，只有在执行完`InstallationMainFun`方法后才能执行。

下面看看`InstallationMainFun`的实现：

	Function InstallationMainFun
	    SendMessage $PB_ProgressBar ${PBM_SETRANGE32} 0 100
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 10 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 20 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 30 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 40 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 50 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 60 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 70 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 80 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 90 0
	    Sleep 1000
	    SendMessage $PB_ProgressBar ${PBM_SETPOS} 100 0
	                                   
	    ShowWindow $Btn_Next ${SW_SHOW}
	    ShowWindow $Btn_Install ${SW_HIDE}
	    EnableWindow $Btn_Cancel 0
	FunctionEnd
所做的工作主要是操作滚动条的位置。在实际的释放过程中，这个设置也是这样的，目前还没有自定义页面的释放插件，能回调得到精确的释放量，以及释放文件等等信息。所以只能模拟，尽可能的细化下来，估计着大概多少。这个是一个需要情商的工作，如果你有洁癖，偏执之类的病，我想你还是用系统自带的释放好了，也不需要自定义页面了 。

最后上几张图吧，好看点，也算是阶段的结束 。

**（安装过程）**

!["安装过程"][2]

**（安装过程结束）**

!["安装过程结束"][3]

**（安装结束）**

!["安装结束"][4]


[1]: https://github.com/nicecai/nsissource/tree/master/4 "例子下载"
[2]: http://m1.img.libdd.com/farm4/2012/1123/00/EB4E0179E1715F0AB4E0B354AC1FCD44E54B5981D32F7_520_348.JPEG "安装过程"
[3]: http://m1.img.libdd.com/farm4/2012/1123/00/DD53AE55D1C134F2DBA34352B3CBDE2A99D5FA1612537_518_348.JPEG "安装过程结束"
[4]: http://m2.img.libdd.com/farm5/2012/1123/00/1B06AB3E30D4DB314BDBC49639B2D36D50F87B4D70553_518_346.JPEG "安装结束"