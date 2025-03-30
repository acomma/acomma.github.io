---
title: QQ 截图启动器
date: 2025-03-29 19:42:58
updated: 2025-03-29 19:42:58
tags: [C#, WinForms]
---

[看雪](https://bbs.kanxue.com/)论坛 [0xEEEE](https://bbs.kanxue.com/homepage-901761.htm) 在[逆向调用QQ截图NT与WeChatOCR](https://bbs.kanxue.com/thread-278161.htm)这篇文章中发布了调用新版 QQ 截图功能的软件 *QQScreenShotNT-Plus*，虽然已经设置了 *QQScreenshot.exe* 的路径，但是每次启动这个软件都要提示是否重新自动获取，在设置中去掉“启动提示”依然无效。在那篇文章中作者说 *QQScreenShotNT-Plus* 是在 [QQImpl](https://github.com/EEEEhex/QQImpl) 基础上实现的，但是作者没有发布对应的源代码，因此我准备基于 [Windows Forms](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/?view=netdesktop-8.0) 技术复刻相关的功能。因为是第一次使用 [.NET](https://learn.microsoft.com/zh-cn/dotnet/) 相关技术开发 Windows 系统的应用软件，所以会详细地记录整个开发过程。源代码在 [QQScreenShotLauncher](https://github.com/acomma/QQScreenShotLauncher)。

<!-- more -->

## 创建解决方案

这部分内容参考[项目和解决方案简介](https://learn.microsoft.com/zh-cn/visualstudio/get-started/tutorial-projects-solutions?view=vs-2022)。启动 Visual Studio 2022，在 *Visual Studio 2022* 对话框选择 *Create a new project*

{% asset_img create-solution-1.png %}

在 *Create a new project* 对话框的在搜索框输入 *solution*，选择*语言*、*平台*、*项目类型*，然后选择 *Blank Solution*

{% asset_img create-solution-2.png %}

在 *Configure your new project* 对话框输入解决方案名称 *QQScreenShotLauncher*，保存位置 *D:\projects\dotnet\\*

{% asset_img create-solution-3.png %}

## 创建 Git 仓库

这部分内容参考[从 Visual Studio 创建 Git 存储库](https://learn.microsoft.com/zh-cn/visualstudio/version-control/git-create-repository?view=vs-2022)。在 Git 变更选择 *Create Git Repository*

{% asset_img git-1.png %}

在 *Create a Git repository* 对话框选择 *Local only*，*Local path*、*.gitignore template*、*License template* 三项使用默认值，勾选 *Add a README.md* 复选框

{% asset_img git-2.png %}

## 创建项目

这部分内容参考[项目和解决方案简介](https://learn.microsoft.com/zh-cn/visualstudio/get-started/tutorial-projects-solutions?view=vs-2022)。在*解决方案资源管理器*中，右键单击 *QQScreenShotLauncher* 解决方案，依次选择 *Add* 和 *New Project*

{% asset_img create-project-1.png %}

在搜索框输入 *Windows Forms App*，选择*语言*、*平台*、*项目类型*，然后选择 *Windows Forms App*

{% asset_img create-project-2.png %}

在 *Configure your new project* 对话框输入项目名称 *NTLauncher*，保存位置保持默认值

{% asset_img create-project-3.png %}

在 *Additional information* 对话框选择框架版本 *.NET 8.0 (Long Term Support)*

{% asset_img create-project-4.png %}

## 添加图标资源

这部分参考[管理应用程序资源](https://learn.microsoft.com/zh-cn/visualstudio/ide/managing-application-resources-dotnet?view=vs-2022)。在*解决方案资源管理器*中，右键单击 *NTLauncher* 项目，选择 *Properties*

{% asset_img add-icon-1.png %}

选中 *Resources > General*，点击 *Create or open assembly resources* 链接

{% asset_img add-icon-2.png %}

在 *Resource Explorer* 点击*加号*按钮

{% asset_img add-icon-3.png %}

在 *Add a new resource* 对话框，*Type* 下拉框选择 *Icon*，其他保持默认，然后点击 *Add existing file* 按钮

{% asset_img add-icon-4.png %}

在文件选择对话框中选择 *NTLauncher.ico*（这个文件来自 [0xEEEE](https://bbs.kanxue.com/homepage-901761.htm) 在[逆向调用QQ截图NT与WeChatOCR](https://bbs.kanxue.com/thread-278161.htm) 提供的 *QQScreenShotNT-Lite.zip* 压缩包）

{% asset_img add-icon-5.png %}

在 *Add a new resource* 对话框，*Neutral value* 就是刚刚选择的文件，*Store as* 下拉框选择 *System.Drawing.Icon (Windows Forms)*，其他保持默认

{% asset_img add-icon-6.png %}

## 设置应用程序图标

这部分参考[指定应用程序图标 （Visual Basic， C#）](https://learn.microsoft.com/zh-cn/visualstudio/ide/how-to-specify-an-application-icon-visual-basic-csharp?view=vs-2022)。在*解决方案资源管理器*中，右键单击 *NTLauncher* 项目，选择 *Properties*

{% asset_img add-icon-1.png %}

选中 *Application > Win32 Resources*，点击 *Browse* 按钮

{% asset_img application-icon-1.png %}

在文件选择对话框中选择 *NTLauncher.ico*，注意选择文件所在的目录，这个文件是上一步*添加图标资源*添加到项目的 *Resources* 目录的

{% asset_img application-icon-2.png %}

文件选择完成后 *Icon* 项的值为 *Resources\NTLauncher.ico*

{% asset_img application-icon-3.png %}

## 添加系统托盘和右键菜单

这部分主要参考了以下资料

1. [NotifyIcon 组件概述（Windows 窗体）](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/notifyicon-component-overview-windows-forms?view=netframeworkdesktop-4.8)
2. [如何使用 Windows 窗体 (Windows Forms) NotifyIcon 组件向任务栏添加应用程序图标](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/app-icons-to-the-taskbar-with-wf-notifyicon?view=netframeworkdesktop-4.8)
3. [如何：将快捷菜单与 Windows 窗体 NotifyIcon 组件相关联](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/how-to-associate-a-shortcut-menu-with-a-windows-forms-notifyicon-component?view=netframeworkdesktop-4.8)
4. [如何将 ContextMenuStrip 与控件关联](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/how-to-associate-a-contextmenustrip-with-a-control?view=netframeworkdesktop-4.8)
5. [如何：向 ContextMenuStrip 添加菜单项](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/how-to-add-menu-items-to-a-contextmenustrip?view=netframeworkdesktop-4.8)
6. [NotifyIcon 类](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.forms.notifyicon?view=windowsdesktop-8.0)
7. [ContextMenuStrip 类](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.forms.contextmenustrip?view=windowsdesktop-8.0)
8. [winform实现最小化至系统托盘](https://www.cnblogs.com/mingupupu/p/18022142)
9. [C#实现 Winform 程序在系统托盘显示图标 & 开机自启动](https://www.cnblogs.com/vipsoft/p/18665897)
10. [WinForm窗体隐藏任务栏图标和系统托盘显示图标](https://blog.csdn.net/zz81238919/article/details/136168347)

### 添加 NotifyIcon 和 ContextMenuStrip 控件

在 *Toolbox* 搜索 *NotifyIcon*，将 *NotifyIcon* 控件拖入 *Form1*

{% asset_img controls-1.png %}

对 *ContextMenuStrip* 控件进行类似的操作，控件添加完成后

{% asset_img controls-2.png %}

### 配置 NotifyIcon 图标

选中 *notifyIcon1* 控件，在 *Properties* 找到 *Appearance > Icon*，点击右侧 *...* 按钮

{% asset_img notify-icon-1.png %}

在文件选择对话框中选择 *NTLauncher.ico*，注意选择文件所在的目录，这个文件是在*添加图标资源*添加到项目的 *Resources* 目录的

{% asset_img application-icon-2.png %}

### 配置 NotifyIcon 右键菜单

选中 *notifyIcon1* 控件，在 *Properties* 找到 *Behavior > ContextMenuStrip*，在右侧下来选项中选择 *contextMenuStrip1*

{% asset_img notify-icon-2.png %}

### 配置 NotifyIcon 其他属性

选中 *notifyIcon1* 控件，配置 *Appearance > Text* 为 *NTLauncher*，配置 *Behavior > Visible* 为 *True*

{% asset_img notify-icon-3.png %}

*notifyIcon1* 的 *Visible* 配合 *Form1* 的 *ShowInTaskbar* 和 *WindowState* 就可以实现启动应用程序只显示托盘图标的功能。

### 添加右键菜单项

选中 *contextMenuStrip1* 控件，点击右上角三角形图标，在弹出的 *ContextMenuStrip Tasks* 中点击 *Edit Items*

{% asset_img context-menu-item-1.png %}

在 *Items Collection Editor* 对话框选中 *contextMenuStrip1* 成员，选择 *MenuItem*，点击 *Add* 按钮添加菜单项

{% asset_img context-menu-item-2.png %}

在 *Items Collection Editor* 对话框选中 *contextMenuStrip1* 成员，选择 *Separator*，点击 *Add* 按钮添加分割线

{% asset_img context-menu-item-3.png %}

我们将添加 7 个菜单项，3 条分割线，通过右侧的功能按钮*排序*或*删除*

{% asset_img context-menu-item-4.png %}

选中 *toolStripMenuItem1*，修改 *Appearance > Text* 的值为*关于*

{% asset_img context-menu-item-5.png %}

对其他菜单项进行类似操作，编辑完成后的 *Form1*

{% asset_img context-menu-item-6.png %}

### 使应用程序只显示托盘图标

选中 *Form1* 控件，设置 *Layout > WindowState* 为 *Minimized*，设置 *Window Style > ShowInTaskbar* 为 *False*

{% asset_img form1-1.png %}

### 添加托盘退出事件

选中 *contextMenuStrip1* 控件，然后选中*退出*菜单项，配置 *Action > Click* 为 *toolStripMenuItem7_Click*

{% asset_img context-menu-item-action-1.png %}

上面的操作会自动在 `Form1.cs` 中创建函数 `toolStripMenuItem7_Click`，其实现为

```csharp
private void toolStripMenuItem7_Click(object sender, EventArgs e)
{
    Application.Exit();
}
```

## 添加设置对话框

这部分内容主要参考了以下资料

1. [C# Winform同一子窗体只允许打开一次](https://www.cnblogs.com/baissy/p/12689287.html)
2. [WinForm窗体应用——父窗体每次只打开一个子窗体的方法](https://www.cnblogs.com/hanzq/p/16978697.html)
3. [c# WinForm 点击出现弹出多个窗体， 怎么才能只显示一个窗体。解决方案](https://blog.csdn.net/qq_28821897/article/details/121549310)
4. [C# WinForm 点击按钮显示唯一窗体](https://blog.csdn.net/qq_43022415/article/details/106667387)
5. [C#窗体程序（winform）禁止最小化、最大化，或去掉关闭按钮](https://blog.csdn.net/feiduan1211/article/details/88419561)
6. [WinForm 设置窗体启动位置在活动屏幕右下角](https://www.cnblogs.com/aning2015/p/9268486.html)

### 新增设置对话框

在 *NTLauncher* 项目上单击右键，在弹出菜单中依次选择 *Add > New Items*

{% asset_img form2-1.png %}

在 *Add New Item* 对话框中选中 *Installed > C# Items*，在右侧选中 *Form (Windows Forms)*，注意下方 *Name* 的值 *Form2.cs*

{% asset_img form2-2.png %}

### 点击设置菜单项显示设置对话框

在 *Form1* 选中 *contextMenuStrip1* 控件，然后选中*设置*菜单项，配置 *Action > Click* 为 *toolStripMenuItem2_Click*

{% asset_img context-menu-item-action-2.png %}

上面的操作会自动在 `Form1.cs` 中创建函数 `toolStripMenuItem2_Click`，其实现为

```csharp
private Form2? form2 = null;

private void toolStripMenuItem2_Click(object sender, EventArgs e)
{
    if (form2 == null || form2.IsDisposed)
    {
        form2 = new Form2();
        form2.FormClosed += (s, _) =>
        {
            form2?.Dispose(); // 显式释放资源
            form2 = null;     // 强制置空
        };
        form2.ShowDialog();
    }
    else
    {
        form2.Activate();
    }
}
```

在这个实现里增加了一个变量 `form2` 来表示打开的设置对话，对话框关闭时把该变量置空，这保证多次点击*设置*菜单只打开一个对话框。

### 配置设置对话框样式

选中 *Form2*，设置 *Appearance > Text* 为 *设置*，设置 *Window Style > MaximizeBox* 为 *False*，设置 *Window Style > MinimizeBox* 为 *False*，设置 *Window Style > ShowIcon* 为 *False*，设置 *Window Style > ShowInTaskbar* 为 *False*

{% asset_img form2-3.png %}

### 设置对话框启动位置在屏幕右下角

选中 *Form2*，设置 *Layout > StartPosition* 为 *Manual*

{% asset_img form2-4.png %}

在 `Form2.cs` 中重新计算对话框的位置

```csharp
public Form2()
{
    InitializeComponent();

    Screen screen = Screen.FromPoint(new Point(Cursor.Position.X, Cursor.Position.Y));
    int x = screen.WorkingArea.X + screen.WorkingArea.Width - this.Width;
    int y = screen.WorkingArea.Y + screen.WorkingArea.Height - this.Height;
    this.Location = new Point(x, y);
}
```

## 绘制对话框

这部分内容主要参考了以下资料

1. [教程：使用 .NET 创建 Windows 窗体应用](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/get-started/create-app-visual-studio?view=netdesktop-8.0)
2. [教程：创建数学测验 WinForms 应用](https://learn.microsoft.com/zh-cn/visualstudio/get-started/csharp/tutorial-windows-forms-math-quiz-create-project-add-controls?view=vs-2022)
3. [控件的位置和布局（Windows 窗体 .NET）](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/layout?view=netdesktop-8.0)
4. [演示：使用捕捉线排列 Windows 窗体上的控件](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/walkthrough-arranging-controls-on-windows-forms-using-snaplines?view=netframeworkdesktop-4.8)
5. [如何：在 Windows 窗体上对齐多个控件](https://learn.microsoft.com/zh-cn/dotnet/desktop/winforms/controls/how-to-align-multiple-controls-on-windows-forms?view=netframeworkdesktop-4.8)
6. [Winform禁止调整大小](https://blog.csdn.net/csdn_wuwt/article/details/88222159)

主要用到的控件为 *GroupBox*、*CheckBox*、*Label*、*TextBox*、*Button*，成品图如下所示

{% asset_img form2-5.png %}

选中 *Form2*，设置 *Appearance > FormBorderStyle* 为 *FixedSingle*，禁止改变窗口大小

{% asset_img form2-6.png %}
