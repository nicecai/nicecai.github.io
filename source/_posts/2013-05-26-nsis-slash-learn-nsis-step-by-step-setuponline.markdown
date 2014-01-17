---
layout: post
title: "NSIS进阶教程(五)"
date: 2013-05-26 12:24
comments: true
categories: NSIS
---

### 在线下载，下载后的安装，本地解压安装 ###

## 前言 ##

>
想做在线安装包教程已经很久了，一直找不到合适的时间，如今在一个慵懒的周末早晨终于开始了。今天晚上还要去虹口体育场看申花跟国安的比赛，还是要抓紧时间呀。
在线安装是以前教程的一个延续。
>
**`意义一：`**国内很多的IT厂商在做安装包的时候首先考虑的是用户下载的时间可接受度，而一个动辄100mb的包会把用户吓跑，而且下载的入口还控制在诸如迅雷，旋风，IE，等等，稍等，还有360安全浏览器。。。你如果没有给360打个招呼，它说不定会报你的程序有毒。。。太可拍了。入口最重要，掌握在任何人手里都会成为收门票的工具。
>
**`意义二：`**用户对着迅雷，旋风的界面，千遍一律，没有任何信息的展示，就是一个进度条，你丧失了在第一时间占领用户意识的契机。如果你在安装的时候有动画播放，并且让用户在安装的过程中获取产品的使用技巧，或者像广告植入一样的严肃的对待这个不起眼的时间，你会收获很多。

## 本篇主要讲讲以下几点： ##

+ **在线下载**

+ **下载后的安装**

+ **下载后的解压**

**所用到的插件【新增】：**

1. inetc
2. nsis7z

<!--more-->

**讲义**

**首先贴出今天教程的完整的例子【已测试】【附带图片】 [猛击这里][1]**
>
题外话:最终还是决定把所有源码转移到Github上面，虽然前段时间也被GFW给封了一下，但是还是不能影响我对Git以及开源项目的热情。我相信开放的才是最光明的。

*这次教程主要处理在线安装的问题，所谓的在线安装，必须要有下载和安装两部分组成，安装又分为执行或者解压两个部分，这次一次性的把安装跟解压都跟大家探讨一下*

## 在线下载 ##
我摸索了一下`NSIS`的插件，最终确定用`Intec`，其他的也都看过，不是功能不适合就是我没有找到源码，选择`Intec`其实也是勉强而为之，因为它的3个下载的样式都不适合自定义界面的要求，因为它的进度条，它的百分比都是自己已经全部包装好了的。对着丑的惨不忍睹的界面，只有叹气的份了。

怎么办？

只能改源码，打开`Intec`插件的**[`C++`源码][2]**，修改两点：

1. 把它`Dialog`中的`ProgressBar`给隐藏掉，或者你直接把该`Dialog`给`Hide`掉也可以。
2. 修改其中的传值，如果你的自定义页面`CustomPage`中需要有多处地方需要统一的显示百分比进度的话需要在调用`Intec`的时候多传一些控件进去。你就可以在代码中修改

因为我需要的只是一个进度的返回，以及`ProgressBar`的反应，所以所有的`Intec`的界面都被我`Hide`掉了，这里就不再贴源码了，因为我也找不到了，只找到当初`Release`出来的一个`Dll`，在给的源码包中。所有的传值方式都跟官方原版中的Intec是一样的，只是在参数中多加了几个外部组件的传入入口。

By the way想改源码的童鞋，这个Intec的源码必须在VC6下编译，VC2005及以上编译出来的在有些平台下会有异常。
嫌麻烦的话直接用给的也可以，照着例子中的调用即可。

上代码：

	inetc::get /hwnd $PPercent /hwnd2 $PPercent /probar $PB_ProgressBar /caption " " /popup "" "http://download.microsoft.com/download/c/6/e/c6e88215-0178-4c6c-b5f3-158ff77b1f38/NetFx20SP2_x64.exe" "$TEMP\NetFx20SP2_x64.exe"  /end
	Pop $0
	${If} $0 == "Transfer Error"
	  ;MessageBox MB_OK "net64err"
		Call onClickClose
		Abort
	${ELSEIF} $0 == "SendRequest Error"
		;MessageBox MB_OK "net64err2"
		Call onClickClose
		Abort
	${ELSE}
		;System::Call "user32::InvalidateRect(i $hwndparent,i0,i 1)"
		Call NFInstall64
	${EndIf}

其中`/hwnd` `/hwnd2` `/probar`都是自定义界面上面的组件传入参数，后面紧跟的都是自己创建的`Label`跟`Progress`:`$PPercent` `$PB_ProgressBar`

下载完成后有返回值一般等于`OK`即可，但是经测试不能用唯一正确的`OK`来判别，因为有可能下载正确但是返回值不是`OK`，我就用了两个最严重的错误来判别，其他的就算是下载正确了。
下载到`$TEMP`文件夹中，该文件夹在系统盘临时目录中，你测试的时候也可以直接设定一个目录，查看下载情况。
下载就说到这里。

## 下载后的安装 ##
这里的安装当然是指譬如微软的msi，微软的exe，或者是某些可执行程序。如果你需要执行这些后缀的程序的话，最好先了解一下该执行程序所依靠的环境。例子中是用`.net framework 2.0 sp2`做的例子，该执行程序需要依靠`Microsoft Installer 3.1`的安装环境，所以例子中在这一点也做了检测，譬如安装该执行程序所需的硬盘空间，在例子中也有体现，详情可以看代码。

`.net framework 2.0 sp2`的安装一般人都不希望弹出对话框出来，所以需要静默安装。

	nsExec::ExecToStack '"$TEMP\NetFx20SP2_x64.exe" /q /c:"install.exe /noaspupgrade /q"'
	Pop $0
	${If} $0 == "timeout"
		Call onClickClose
		Abort
	${EndIf}
	;MessageBox MB_OK "net64inst2"
	${If} $0 == "error"
	  	;MessageBox MB_OK "net64inst1111111"
		Call onClickClose
		Abort
	${EndIf}

	Delete "$TEMP\NetFx20SP2_x64.exe"
参数`/q /c:"install.exe /noaspupgrade /q"`是该执行程序的静默安装方式。
Pop出来的$0是安装结果，只有没有安装过的机器才能安装，安装过的机器都会报错，所以在安装.net framework之前需要读注册表判断一下是否安装过，例子中有。

	Function NFEnvironmentCheck
		ReadRegStr $1 HKLM "Hardware\Description\System\CentralProcessor\0" Identifier
		StrCpy $2 $1 3
		${If} $2 == 'x86'
			ReadRegStr $7 HKLM 'SOFTWARE\Microsoft\Windows NT\CurrentVersion' CurrentVersion
			ReadRegStr $8 HKLM 'SOFTWARE\Microsoft\NET Framework Setup\NDP\v2.0.50727' Install
			ReadRegStr $9 HKLM 'SOFTWARE\Microsoft\NET Framework Setup\NDP\v2.0.50727' SP
		${Else}
			SetRegView 64
			ReadRegStr $7 HKLM 'SOFTWARE\Microsoft\Windows NT\CurrentVersion' CurrentVersion
			ReadRegStr $8 HKLM 'SOFTWARE\Microsoft\NET Framework Setup\NDP\v2.0.50727' Install
			ReadRegStr $9 HKLM 'SOFTWARE\Microsoft\NET Framework Setup\NDP\v2.0.50727' SP
			SetRegView lastused
		${EndIf}
	
		${If} $7 == '6.2'	;Win8
		${OrIf} $7 == '6.1'	;Win7
			;判断是否装了.net framework 2.0 sp2
			${If} $8 == 1
			${AndIf} $9 == 2
	
			${Else}
				;GetFunctionAddress $0 DisplayNFLabel
				;BgWorker::CallAndWait
				${If} $2 == 'x86'
					StrCpy $ISNETF 1
				${Else}
				  StrCpy $ISNETF 1
				${EndIf}
			${EndIf}
	
		${ElseIf} $7 == '6.0' ;WinVista
		${OrIf} $7 == '5.2'  ;Win2003
		${OrIf} $7 == '5.1'  ;WinXP
		${OrIf} $7 == '5.0'  ;Win2000
			;判断是否装了.net framework 2.0 sp2
			${If} $8 == 1
			${AndIf} $9 == 2
	
			${Else}
				${If} $2 == 'x86'
					StrCpy $ISNETF 1
				${Else}
				  StrCpy $ISNETF 1
	      ${EndIf}
			${EndIf}
		${EndIf}
	FunctionEnd

最后删除`TEMP`文件夹中的临时文件
整个安装过程就是这样的了。

## 下载后的解压 ##
这一部分所需要解决的问题就是当用户需要下载自己的程序进行安装的时候，又不能自己再做一个msi的安装包吧，这样可以但是很麻烦，于是乎就直接用压缩的方式吧，这样方便。放眼压缩界7z绝对算是一个奇葩，有人做过统计，7z的压缩比是综合最好的，虽然压缩的时候比其他的压缩算法慢，但是解压却是差不多。幸好NSIS也有7z的[插件][3]
在所有工作准备好之前，你需要把你的程序通过客户端自己的7z开源软件压缩好，并且放在一个服务器上面以供下载之用。
例子中，我直接用的是7z官方的地址，作为我需要安装的软件。
下载跟之前的是一样的，这里只说解压：

	SetOutPath $INSTDIR
	SetOverwrite on
	Nsis7z::Extract "$TEMP\7z920_extra.7z"
	Delete "$TEMP\7z920_extra.7z"

把解压的路径定位于你之前设定的安装路径`$INSTDIR`中，调用`Nsis7z::Extract`命令进行解压操作，最后删除TEMP中的临时文件。
过程非常简单。

**结束语**
在线安装是一个趋势，程序的下载也需要专门的工具去控制，寄希望于其他软件的高性能表现还不如自己动手把该做的事情都做了。
如果哪位朋友要做一些很酷的动画用于在线安装程序中，那得好好研究一下NSIS的插件制作，用C++做一个插件出来奉献给大家最好了。

最近发现猎豹浏览器的安装程序不错值得模仿一下哦。

上图：


![安装开始](/static/nsis_51.png)

![安装过程](/static/nsis_52.png)


[1]: https://github.com/nicecai/nsissource/tree/master/5 "例子下载"
[2]: http://nsis.sourceforge.net/mediawiki/images/c/c9/Inetc.zip "Intec源码"
[3]: http://nsis.sourceforge.net/Nsis7z_plug-in "7z插件"