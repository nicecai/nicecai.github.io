---
layout: post
title: "NSIS进阶教程(三)"
date: 2013-05-26 11:09
comments: true
categories: NSIS
---

### 自定义MessageBox，自定义页跳转，自定义CheckBox样式 ###

## 前言 ##
>
上一节中我们处理了Button的自定义以及Button的事件消息、协议框的创建等等，这节中我们要更加完美的要求我们的提示框也要漂亮，CheckBox也要自定义样式。有人说MessageBox在NSIS默认情况下是带边框的API窗口，是一个比较丑雏形，但是NSIS的nsDialogs插件也没有提供一个可以创建弹出窗的命令行呀，CheckBox系统自带的只能是那个默认的皮肤，如果想把CheckBox的背景改成蓝色，或者把CheckBox的勾改成爱心，把叉改成骷髅头如何去做呢？这些问题在任何一个面向对象的编程语言中都可以很容易实现，在NSIS这样局限的环境中也可以变通实现，接下来我们处理一下这些问题。

## 本篇主要讲讲以下几点： ##

+ **如何抛弃系统的MessageBox**

+ **如何实现自定义页面的跳转**

+ **如何用自己的贴图实现CheckBox功能**

**所用到的插件【新增】：**

1. FindProcDLL（查找当前进程插件）

2. KillProcDLL （关闭指定进程插件）

<!--more-->
**讲义**

**首先贴出一个今天教程的完整的例子【已测试】【附带图片】 [猛击这里][1]**

*题外话，本节的重点是如何把系统的MessageBox替换掉自定义的窗口，一开始这个命题挺难的，原本因为nsDialogs可以创建一个弹窗，然后给它披上一层皮就可以了，在查了很多资料后，我了解到根本不行，nsDialogs插件不能完成这么简单的工作，而是通过另一个插件nsWindows来完成。另一个自定义CheckBox功能还是有点讨巧的，其实在窗体编程中，Button跟CheckBox本质上是一样的东西，CheckBox就是Button披上一层皮加上一些单击事件改变皮肤而存在的，于是就有了CheckBox的自定义的可能性。*

*还是要讲一点不足，此处CheckBox不包含旁边的说明文字，也就是说，当你促发CheckBox旁边的文字某些事件的时候CheckBox本事不会有任何反应，这样就不能构成一个CheckBox的整体，不过对于要求不是非常高的人来说，这点完全可以忽略。*

## 如何抛弃系统的MessageBox ##

通常在使用警告框的时候，我们会调用MessageBox的NSIS命令，然而此窗口无比丑陋，简直是对我们正在做的窗体的亵渎。于是我们自己创建警告框，此警告框也要跟我们已有的窗口风格一致，包含几大问题，“无边框”、“美观贴图”、“移动”、“逻辑判断功能”，做到以上几点就可以了。上代码：

	Function onCancel
	    IsWindow $WarningForm Create_End
	    !define Style ${WS_VISIBLE}|${WS_OVERLAPPEDWINDOW}
	    ${NSW_CreateWindowEx} $WarningForm $hwndparent ${ExStyle} ${Style} "" 1018
	                                                                                                                               
	    ${NSW_SetWindowSize} $WarningForm 349 184
	    EnableWindow $hwndparent 0
	    System::Call `user32::SetWindowLong(i$WarningForm,i${GWL_STYLE},0x9480084C)i.R0`
	    ${NSW_CreateButton} 148 122 88 25 ''
	    Pop $R0
	    StrCpy $1 $R0
	    Call SkinBtn_Quit
	    ${NSW_OnClick} $R0 OnClickQuitOK
	                                                                                                                               
	    ${NSW_CreateButton} 248 122 88 25 ''
	    Pop $R0
	    StrCpy $1 $R0
	    Call SkinBtn_Cancel
	    ${NSW_OnClick} $R0 OnClickQuitCancel
	                                                                                                                               
	    ${NSW_CreateBitmap} 0 0 100% 100% ""
	    Pop $BGImage
	  ${NSW_SetImage} $BGImage $PLUGINSDIR\quit.bmp $ImageHandle
	    GetFunctionAddress $0 onWarningGUICallback
	    WndProc::onCallback $BGImage $0 ;处理无边框窗体移动
	  ${NSW_CenterWindow} $WarningForm $hwndparent
	    ${NSW_Show}
	    Create_End:
	  ShowWindow $WarningForm ${SW_SHOW}
	FunctionEnd
这个`onCancel`方法是在我们在主窗体上点击“取消”或者“关闭”的时候促发的方法，这时会弹出一个命令窗口出来。

创建跟设置样式：`WS_VISIBLE`  是显示的意思，不加会隐藏掉，`WS_OVERLAPPEDWINDOW`  是创建一个层叠窗体的属性也需要加上。`NSW_CreateWindowEx` 是创建窗体的主命令。

创建完窗体后，用`NSW_SetWindowSize`  更改一下窗体的大小。

`EnableWindow $hwndparent 0`这段代码一定要加上，这是创建一个警告框模式窗体的必要条件，就是让主窗体不能操作。

接下来就是消除边框，用`nsWindows`命令创建按钮的一套，跟`nsDialogs`的创建是一致的，包括无边框移动的一套，在第一节中已经讲过，这里就不啰嗦了。

最后把这个窗体展示出来就可以了。把这个`onCancel`这个方法赋给“取消”按钮的单击事件，这样一个无边框模式窗体警告框就好了。

!["自定义弹出框"][2]

当我们点击“退出”的时候，就要关掉当前程序，所以要添加一个关闭的方法：

	Function onClickClose
	    FindProcDLL::FindProc "test.exe"
	    Sleep 500
	    Pop $R0
	    ${If} $R0 != 0
	    KillProcDLL::KillProc "test.exe"
	    ${EndIf}
	FunctionEnd
通过`FindProcDLL`插件的`FindProc`方法找到安装进程，并且通过`KillProcDLL`插件的`KillProc`杀之。

当我们点击“取消”的时候，这时候需要关闭当前警告框，并且让主窗体能够处于Active状态

	Function OnClickQuitCancel
	  ${NSW_DestroyWindow} $WarningForm
	  EnableWindow $hwndparent 1
	  BringToFront
	FunctionEnd
`NSW_DestroyWindow`命令销毁掉警告框，使主窗体能活动，并且`Bring`到前端。

## 如何实现自定义页面的跳转 ##

一开始我们就定义了两个自定义页面：

	Page custom WelcomePage
	Page custom InstallationPage
从`WelcomePage`跳转到`InstallationPage`，这个就是`下一步`按钮的事件。

创建一个`RelGotoPage`方法：

	Function RelGotoPage
	  IntCmp $R9 0 0 Move Move
	    StrCmp $R9 "X" 0 Move
	      StrCpy $R9 "120"
	  Move:
	  SendMessage $HWNDPARENT "0x408" "$R9" ""
	FunctionEnd
详细解释在这里 [http://nsis.sourceforge.net/Go_to_a_NSIS_page][3]
>
If a number > 0: Goes foward that number of pages. Code of that page will be executed, not returning to this point. If it is bigger than the number of pages that are after that page, it simulates a "Cancel" click.
>
If a number < 0: Goes back that number of pages. Code of that page will be executed, not returning to this point. If it is bigger than the number of pages that are before that page, it simulates a "Cancel" click.
>
If X: Simulates a "Cancel" click. Code will go to callback functions, not returning to this point.
>
If 0: Continues on the same page. Code will still be running after the call.

在`下一步`的单击事件中加入以下代码，跳转就好了，而且以后的调转RelGotoPage都适用

	Function onClickNext
	  StrCpy $R9 1
	  Call RelGotoPage
	  Abort
	FunctionEnd


## 如何用自己的贴图实现CheckBox功能 ##

`CheckBox`也有系统自带的那种健全的，但是效果没有`Button`贴皮肤后好，所以弃用之，在第二个页面中创建几个`Button`跟标签：
	
	${NSD_CreateButton} 26 150 15 15 ""
	    Pop $Ck_ShortCut
	    StrCpy $1 $Ck_ShortCut
	    Call SkinBtn_Checked
	    GetFunctionAddress $3 OnClick_CheckShortCut
	    SkinBtn::onClick $1 $3
	    StrCpy $Bool_ShortCut 1
	    ${NSD_CreateLabel} 45 151 100 15 "添加桌面快捷方式"
	    Pop $Lbl_ShortCut
	    SetCtlColors $Lbl_ShortCut ""  transparent ;背景设成透明
创建`CheckBox`的时候要考虑周全，首先要定义该`CheckBox`的变量，该`CheckBox`的皮肤，记录该`CheckBox`状态的变量`$Bool_ShortCut`，该`CheckBox`旁边的提示文字变量`$Lbl_ShortCut`

!["自定义Checkbox"][4]
               
!["自定义Checkbox"][5]

Button的贴图跟变换方式在第二节中已经介绍，这里就不啰嗦。

这里的`OnClick_CheckShortCut`方法不仅仅是一个变换的实现，也通过`$Bool_ShortCut`记录了当前`CheckBox`的状态

	Function OnClick_CheckShortCut
	  ${IF} $Bool_ShortCut == 1
	        IntOp $Bool_ShortCut $Bool_ShortCut - 1
	        StrCpy $1 $Ck_ShortCut
	        Call SkinBtn_UnChecked
	    ${ELSE}
	        IntOp $Bool_ShortCut $Bool_ShortCut + 1
	        StrCpy $1 $Ck_ShortCut
	        Call SkinBtn_Checked
	    ${EndIf}
	FunctionEnd
最后我们在完成安装的时候，可以通过`$Bool_ShortCut`该变量来了解用户的选择

!["整体图"][6]

**结束语**

其实不难，逻辑清晰，知道自己朝哪个方向去寻找答案，一切就会云开雾散。希望大家能学到一些。

\***************************************************************************

有任何疑问请留言。

That's all

下回继续探讨

[1]: https://github.com/nicecai/nsissource/tree/master/3 "例子下载"
[2]: http://m3.img.libdd.com/farm4/192/463CC65762D1D65071C86536FAE31FC0_511_350.JPEG "自定义弹出框"
[3]: http://nsis.sourceforge.net/Go_to_a_NSIS_page "gotoansispage"
[4]: http://m2.img.libdd.com/farm5/21/746DB83C38C5A17BDD21506D6D3DF615_15_75.jpg "自定义Checkbox"
[5]: http://m1.img.libdd.com/farm5/248/604441F84638FEF06106931B3A6A6EF8_15_75.jpg "自定义Checkbox"
[6]: http://m2.img.libdd.com/farm4/125/5E2EF978988C13590D44C1E66D174D7D_524_348.JPEG "整体图"