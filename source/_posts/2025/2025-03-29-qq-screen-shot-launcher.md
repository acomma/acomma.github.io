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

## 创建配置文件

在 *NTLauncher* 项目上单击右键，在弹出菜单中依次选择 *Add > New Items*

{% asset_img form2-1.png %}

在弹出的 *Add New Item* 对话框中选中 *Installed > C# Items > General*，然后选中 *Text file*，修改 *Name* 处的名称为 *config.ini*

{% asset_img config-1.png %}

文件的内容来自 *QQScreenShotNT-Plus* 的初始配置

```ini
[ExePath]
QQScreenShot=
UsrLib=
WeChatOCR=
WeChatUtility=
[General]
AutoExitClear=false
AutoNTVOpen=false
AutoRun=false
DebugConsole=false
EnableHotKey=false
EnableOCR=false
EnablePlugin=false
EnableScreenShot=false
EnableUtility=false
FisrtRun=true
HotKey=CTRL+ALT+A
KillXPlugin=false
RunTip=true
ScrollVol=false
[NTViewer]
DbgConsole=false
LaunchShow=false
[Plugin]
CurOCR=NULL
CurSearch=.\ntplugin\baidusearch.py
CurSoutu=.\ntplugin\baidusoutu.py
CurTran=.\ntplugin\youdaotran.py
PluginDir=.\ntplugin
PythonDir=.\pyenv
```

参考[在 .NET 项目中复制资源文件夹到生成目录](https://blog.csdn.net/sd7o95o/article/details/136668458)，在 *Solution Explorer* 中选中 *config.ini* 文件，在 *Properties* 中修改 *Advanced > Copy to Output Directory* 为 *Copy always*

{% asset_img config-2.png %}

在 Visual Studio 中通过这种方式创建的文件的格式是 *UTF-8 with BOM*，在使用 *kernel32.dll* 中的 `GetPrivateProfileString`、`WritePrivateProfileString` 读写时无法读写第一节的内容。有两种避免的方式，第一种是参考 [C#读写ini配置文件](https://blog.csdn.net/congduan/article/details/9324113) 和 [INI文件使用时的注意事项](https://blog.csdn.net/DLS756/article/details/109082078)让 *ini* 文件的第一行为空行。第二种是重新创建一个 *UTF-8 without BOM* 格式的文件。这里选择第三种方式，直接使用 *QQScreenShotNT-Plus* 中的 *config.ini* 文件，它是 *UTF-8 without BOM* 格式的文件。

## 读写配置文件

### 添加配置类

在 *NTLauncher* 项目上单击右键，在弹出菜单中依次选择 *Add > New Items*

{% asset_img form2-1.png %}

在弹出的 *Add New Item* 对话框中选中 *Installed > C# Items > Code*，然后选中 *Class*，修改 *Name* 处的名称为 *Config.cs*

{% asset_img config-3.png %}

*Config.cs* 文件的内容如下所示，*config.ini* 中的每一个键对应一个属性

```csharp
namespace NTLauncher
{
    public class Config
    {
        public string QQScreenShot { get; set; }
        public string UsrLib { get; set; }
        public string WeChatOCR { get; set; }
        public string WeChatUtility { get; set; }
        public string AutoExitClear { get; set; }
        public string AutoNTVOpen { get; set; }
        public string AutoRun { get; set; }
        public string DebugConsole { get; set; }
        public string EnableHotKey { get; set; }
        public string EnableOCR { get; set; }
        public string EnablePlugin { get; set; }
        public string EnableScreenShot { get; set; }
        public string EnableUtility { get; set; }
        public string FisrtRun { get; set; }
        public string HotKey { get; set; }
        public string KillXPlugin { get; set; }
        public string RunTip { get; set; }
        public string ScrollVol { get; set; }
        public string DbgConsole { get; set; }
        public string LaunchShow { get; set; }
        public string CurOCR { get; set; }
        public string CurSearch { get; set; }
        public string CurSoutu { get; set; }
        public string CurTran { get; set; }
        public string PluginDir { get; set; }
        public string PythonDir { get; set; }
    }
}
```

### 添加读写配置的工具类

这部分内容参考以下资料

1. [C# 读写编辑INI文件-完整详细版](https://blog.csdn.net/weixin_45499836/article/details/118353614)
2. [c# winform程序读写ini配置文件](https://www.cnblogs.com/falcon-fei/p/9849868.html)
3. [.NET(C#)读写ini配置文件的方法及示例代码](https://www.cnblogs.com/fireicesion/p/16809687.html)
4. [C#读取写入ini配置文件-以winform为例](https://blog.csdn.net/weixin_46086778/article/details/137241827)
5. [使用类和对象探索面向对象的编程](https://learn.microsoft.com/zh-cn/dotnet/csharp/fundamentals/tutorials/classes)

使用 *kernel32.dll* 中的 `GetPrivateProfileString`、`WritePrivateProfileString` 方法读写 *config* 配置文件

```csharp
using System.Runtime.InteropServices;
using System.Text;

namespace NTLauncher
{
    public class ConfigHandler
    {
        private string filePath;

        [DllImport("kernel32.dll")]
        private static extern long WritePrivateProfileString(string section, string key, string val, string filePath);

        [DllImport("kernel32.dll")]
        private static extern int GetPrivateProfileString(string section, string key, string defVal, StringBuilder retVal, int size, string filePath);

        public ConfigHandler(string filePath)
        {
            this.filePath = filePath;
        }

        public void Write(Config config)
        {
            // ExePath
            WritePrivateProfileString("ExePath", "QQScreenShot", config.QQScreenShot, this.filePath);
            WritePrivateProfileString("ExePath", "UsrLib", config.UsrLib, this.filePath);
            WritePrivateProfileString("ExePath", "WeChatOCR", config.WeChatOCR, this.filePath);
            WritePrivateProfileString("ExePath", "WeChatUtility", config.WeChatUtility, this.filePath);

            // General
            WritePrivateProfileString("General", "AutoExitClear", config.AutoExitClear, this.filePath);
            WritePrivateProfileString("General", "AutoNTVOpen", config.AutoNTVOpen, this.filePath);
            WritePrivateProfileString("General", "AutoRun", config.AutoRun, this.filePath);
            WritePrivateProfileString("General", "DebugConsole", config.DebugConsole, this.filePath);
            WritePrivateProfileString("General", "EnableHotKey", config.EnableHotKey, this.filePath);
            WritePrivateProfileString("General", "EnableOCR", config.EnableOCR, this.filePath);
            WritePrivateProfileString("General", "EnablePlugin", config.EnablePlugin, this.filePath);
            WritePrivateProfileString("General", "EnableScreenShot", config.EnableScreenShot, this.filePath);
            WritePrivateProfileString("General", "EnableUtility", config.EnableUtility, this.filePath);
            WritePrivateProfileString("General", "FisrtRun", config.FisrtRun, this.filePath);
            WritePrivateProfileString("General", "HotKey", config.HotKey, this.filePath);
            WritePrivateProfileString("General", "KillXPlugin", config.KillXPlugin, this.filePath);
            WritePrivateProfileString("General", "RunTip", config.RunTip, this.filePath);
            WritePrivateProfileString("General", "ScrollVol", config.ScrollVol, this.filePath);

            // NTViewer
            WritePrivateProfileString("NTViewer", "DbgConsole", config.DbgConsole, this.filePath);
            WritePrivateProfileString("NTViewer", "LaunchShow", config.LaunchShow, this.filePath);

            // Plugin
            WritePrivateProfileString("Plugin", "CurOCR", config.CurOCR, this.filePath);
            WritePrivateProfileString("Plugin", "CurSearch", config.CurSearch, this.filePath);
            WritePrivateProfileString("Plugin", "CurSoutu", config.CurSoutu, this.filePath);
            WritePrivateProfileString("Plugin", "CurTran", config.CurTran, this.filePath);
            WritePrivateProfileString("Plugin", "PluginDir", config.PluginDir, this.filePath);
            WritePrivateProfileString("Plugin", "PythonDir", config.PythonDir, this.filePath);
        }

        public Config Read()
        {
            Config config = new Config();

            // ExePath

            StringBuilder retVal = new StringBuilder(255);
            GetPrivateProfileString("ExePath", "QQScreenShot", "", retVal, 255, this.filePath);
            config.QQScreenShot = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("ExePath", "UsrLib", "", retVal, 255, this.filePath);
            config.UsrLib = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("ExePath", "WeChatOCR", "", retVal, 255, this.filePath);
            config.WeChatOCR = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("ExePath", "WeChatUtility", "", retVal, 255, this.filePath);
            config.WeChatUtility = retVal.ToString();

            // General

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "AutoExitClear", "false", retVal, 255, this.filePath);
            config.AutoExitClear = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "AutoNTVOpen", "false", retVal, 255, this.filePath);
            config.AutoNTVOpen = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "AutoRun", "false", retVal, 255, this.filePath);
            config.AutoRun = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "DebugConsole", "false", retVal, 255, this.filePath);
            config.DebugConsole = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "EnableHotKey", "false", retVal, 255, this.filePath);
            config.EnableHotKey = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "EnableOCR", "false", retVal, 255, this.filePath);
            config.EnableOCR = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "EnablePlugin", "false", retVal, 255, this.filePath);
            config.EnablePlugin = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "EnableScreenShot", "false", retVal, 255, this.filePath);
            config.EnableScreenShot = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "EnableUtility", "false", retVal, 255, this.filePath);
            config.EnableUtility = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "FisrtRun", "true", retVal, 255, this.filePath);
            config.FisrtRun = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "HotKey", "CTRL+ALT+A", retVal, 255, this.filePath);
            config.HotKey = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "KillXPlugin", "false", retVal, 255, this.filePath);
            config.KillXPlugin = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "RunTip", "true", retVal, 255, this.filePath);
            config.RunTip = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("General", "ScrollVol", "false", retVal, 255, this.filePath);
            config.ScrollVol = retVal.ToString();

            // NTViewer

            retVal = new StringBuilder(255);
            GetPrivateProfileString("NTViewer", "DbgConsole", "false", retVal, 255, this.filePath);
            config.DbgConsole = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("NTViewer", "LaunchShow", "false", retVal, 255, this.filePath);
            config.LaunchShow = retVal.ToString();

            // Plugin

            retVal = new StringBuilder(255);
            GetPrivateProfileString("Plugin", "CurOCR", "NULL", retVal, 255, this.filePath);
            config.CurOCR = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("Plugin", "CurSearch", ".\\ntplugin\\baidusearch.py", retVal, 255, this.filePath);
            config.CurSearch = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("Plugin", "CurSoutu", ".\\ntplugin\\baidusoutu.py", retVal, 255, this.filePath);
            config.CurSoutu = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("Plugin", "CurTran", ".\\ntplugin\\youdaotran.py", retVal, 255, this.filePath);
            config.CurTran = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("Plugin", "PluginDir", ".\\ntplugin", retVal, 255, this.filePath);
            config.PluginDir = retVal.ToString();

            retVal = new StringBuilder(255);
            GetPrivateProfileString("Plugin", "PythonDir", ".\\pyenv", retVal, 255, this.filePath);
            config.PythonDir = retVal.ToString();

            return config;
        }
    }
}
```

### 读取配置文件

在 *Form1.cs* 的构造方法中读取配置文件并保存到全局变量中

```csharp
private ConfigHandler handler;
private Config config;

public Form1()
{
    InitializeComponent();

    this.handler = new ConfigHandler(Application.StartupPath + "config.ini");
    this.config = this.handler.Read();
}
```

### 写入配置文件

这部内容参考以下资料

1. [WinForm中父窗体与子窗体之间的相互传值](https://blog.csdn.net/HerryDong/article/details/103004234)
2. [C#开发Winform实现窗体间相互传值](https://www.zhangshengrong.com/p/v710KqRyXM/)
3. [C#跨窗体传值的几种方法分析（很详细）](https://www.cnblogs.com/xh6300/p/6063649.html)

#### 把配置信息传递给对话框

通过构造函数把配置信息传递给对话框。改造 *Form2.cs* 的构造函数，增加 `Config config` 参数，在构造函数里为表单项赋值

```csharp
private Config config;

public Form2(Config config)
{
    InitializeComponent();

    Screen screen = Screen.FromPoint(new Point(Cursor.Position.X, Cursor.Position.Y));
    int x = screen.WorkingArea.X + screen.WorkingArea.Width - this.Width;
    int y = screen.WorkingArea.Y + screen.WorkingArea.Height - this.Height;
    this.Location = new Point(x, y);

    this.config = config;

    this.checkBox1.Checked = this.config.EnableScreenShot == "true" ? true : false;
    this.checkBox2.Checked = this.config.EnableOCR == "true" ? true : false;
    this.checkBox3.Checked = this.config.EnableUtility == "true" ? true : false;
    this.checkBox4.Checked = this.config.AutoRun == "true" ? true : false;
    this.checkBox5.Checked = this.config.ScrollVol == "true" ? true : false;
    this.checkBox6.Checked = this.config.RunTip == "true" ? true : false;
    this.checkBox7.Checked = this.config.LaunchShow == "true" ? true : false;
    this.checkBox8.Checked = this.config.DebugConsole == "true" ? true : false;
    this.checkBox9.Checked = this.config.DbgConsole == "true" ? true : false;
    this.checkBox10.Checked = this.config.KillXPlugin == "true" ? true : false;
    this.checkBox11.Checked = this.config.AutoNTVOpen == "true" ? true : false;
    this.checkBox12.Checked = this.config.AutoExitClear == "true" ? true : false;
    this.checkBox13.Checked = this.config.EnablePlugin == "true" ? true : false;
    this.checkBox14.Checked = this.config.EnableHotKey == "true" ? true : false;

    string[] hotKey = this.config.HotKey.Split('+');

    this.textBox1.Text = this.config.QQScreenShot;
    this.textBox2.Text = this.config.UsrLib;
    this.textBox3.Text = this.config.WeChatOCR;
    this.textBox4.Text = this.config.WeChatUtility;
    this.textBox5.Text = this.config.PythonDir;
    this.textBox6.Text = this.config.PluginDir;
    this.textBox7.Text = hotKey[0];
    this.textBox8.Text = hotKey[1];
    this.textBox9.Text = hotKey[2];
}
```

在点击*设置*菜单时把配置传递给对话框，如下面的第 5 行代码

```csharp
private void toolStripMenuItem2_Click(object sender, EventArgs e)
{
    if (this.form2 == null || this.form2.IsDisposed)
    {
        this.form2 = new Form2(this.config);
        this.form2.FormClosed += (s, _) =>
        {
            this.form2?.Dispose(); // 显式释放资源
            this.form2 = null;     // 强制置空
        };
        this.form2.ShowDialog();
    }
    else
    {
        this.form2.Activate();
    }
}
```

#### 把配置信息传递给主窗口

为 *Form2* 的*确定*按钮添加点击事件。在 *Form2* 选中*确定*按钮，在 *Properties* 修改 *Action > Click* 为 *button6_Click*

{% asset_img form2-7.png %}

在 `button6_Click` 方法中收集表单的值并对 `this.config` 相关属性赋值

```csharp
private void button6_Click(object sender, EventArgs e)
{
    this.config.EnableScreenShot = this.checkBox1.Checked ? "true" : "false";
    this.config.EnableOCR = this.checkBox2.Checked ? "true" : "false";
    this.config.EnableUtility = this.checkBox3.Checked ? "true" : "false";
    this.config.AutoRun = this.checkBox4.Checked ? "true" : "false";
    this.config.ScrollVol = this.checkBox5.Checked ? "true" : "false";
    this.config.RunTip = this.checkBox6.Checked ? "true" : "false";
    this.config.LaunchShow = this.checkBox7.Checked ? "true" : "false";
    this.config.DebugConsole = this.checkBox8.Checked ? "true" : "false";
    this.config.DbgConsole = this.checkBox9.Checked ? "true" : "false";
    this.config.KillXPlugin = this.checkBox10.Checked ? "true" : "false";
    this.config.AutoNTVOpen = this.checkBox11.Checked ? "true" : "false";
    this.config.AutoExitClear = this.checkBox12.Checked ? "true" : "false";
    this.config.EnablePlugin = this.checkBox13.Checked ? "true" : "false";
    this.config.EnableHotKey = this.checkBox14.Checked ? "true" : "false";

    this.config.QQScreenShot = this.textBox1.Text;
    this.config.UsrLib = this.textBox2.Text;
    this.config.WeChatOCR = this.textBox3.Text;
    this.config.WeChatUtility = this.textBox4.Text;
    this.config.PythonDir = this.textBox5.Text;
    this.config.PluginDir = this.textBox6.Text;
    this.config.HotKey = this.textBox7.Text + "+" + this.textBox8.Text + "+" + this.textBox9.Text;

    this.DialogResult = DialogResult.OK;
}
```

在打开对话框的地方判断结果是否为 `DialogResult.OK`，如果是，则把配置信息写回配置文件，如下面代码第 11~14 行所示

```csharp
private void toolStripMenuItem2_Click(object sender, EventArgs e)
{
    if (this.form2 == null || this.form2.IsDisposed)
    {
        this.form2 = new Form2(this.config);
        this.form2.FormClosed += (s, _) =>
        {
            this.form2?.Dispose(); // 显式释放资源
            this.form2 = null;     // 强制置空
        };
        if (this.form2.ShowDialog() == DialogResult.OK)
        {
            this.handler.Write(this.config);
        }
    }
    else
    {
        this.form2.Activate();
    }
}
```

我们 *Form1* 和 *Form2* 共享同一个 *config* 变量，这可能不是一种好的做法！！！

#### 为对话框的取消按钮添加事件

选中 *Form2* 的*取消*按钮，在 *Properties* 中修改 *Action > Click* 为 *button7_Click*

{% asset_img form2-8.png %}

在 `button7_Click` 方法中设置 `this.DialogResult` 为 `DialogResult.Cancel`

```csharp
private void button7_Click(object sender, EventArgs e)
{
    this.DialogResult = DialogResult.Cancel;
}
```

此时*取消*按钮的功能和窗口的关闭按钮一致。

## 添加 QQImpl 动态链接库

从 [MMMojoCall v3.0.0](https://github.com/EEEEhex/QQImpl/releases/tag/v3.0.0) 下载 [x64-Release.zip](https://github.com/EEEEhex/QQImpl/releases/download/v3.0.0/x64-Release.zip)，解压后将 *MMMojoCall.dll* 复制到 *NTLauncher* 项目目录。在 *Solution Explorer* 选中 *MMMojoCall.dll*，在 *Properties* 设置 *Advanced > Copy to Output Directory* 为 *Copy always*

{% asset_img dll-1.png %}

从 [QQImpl](https://github.com/EEEEhex/QQImpl) 下载 [parent-ipc-core-x64.dll](https://github.com/EEEEhex/QQImpl/blob/main/parent-ipc-core-x64.dll) 并将其复制到 *NTLauncher* 项目目录。在 *Solution Explorer* 选中 *parent-ipc-core-x64.dll*，在 *Properties* 设置 *Advanced > Copy to Output Directory* 为 *Copy always*

{% asset_img dll-2.png %}

## 编译 QQImpl 动态链接库

这部分内容参考了以下资料

1. [DUMPBIN 参考](https://learn.microsoft.com/zh-cn/cpp/build/reference/dumpbin-reference?view=msvc-170)
2. [Visual Studio 中的 CMake 项目](https://learn.microsoft.com/zh-cn/cpp/build/cmake-projects-in-visual-studio?view=msvc-170)
3. [用Visual Studio调试CMake项目并生成Visual Studio工程](https://blog.csdn.net/yxstars/article/details/139889358)

从 [QQImpl](https://github.com/EEEEhex/QQImpl) Releases 下载的 *MMMojoCall.dll* 不包含 *qq_mojoipc*，可以通过如下的方式进行验证。在 Visual Studio 菜单栏选择 *View > Terminal* 进入命令提示符

{% asset_img dumpbin-1.png %}

进入 *NTLauncher* 项目所在的目录 *D:\projects\dotnet\QQScreenShotLauncher\NTLauncher* 执行 `dumpbin /EXPORTS .\MMMojoCall.dll` 命令，从结果中未能找到 *qq_mojoipc* 相关的内容。对 *QQScreenShotNT-Plus* 使用的 *MMMojoCall.dll* 执行相同的命令，从结果中可以找到 *qq_mojoipc* 相关的内容，下面是节选的部分内容

```text
         20   13 00015A70 ?ConnectedToChildProcess@QQIpcParentWrapper@qqipc@qqimpl@@QEAA_NH@Z
         27   1A 00015B40 ?GetLastErrStr@QQIpcChildWrapper@qqipc@qqimpl@@QEAAPEBDXZ
         28   1B 00015B40 ?GetLastErrStr@QQIpcParentWrapper@qqipc@qqimpl@@QEAAPEBDXZ
         31   1E 00015BB0 ?InitEnv@QQIpcChildWrapper@qqipc@qqimpl@@QEAA_NPEBD@Z
         32   1F 00015D40 ?InitEnv@QQIpcParentWrapper@qqipc@qqimpl@@QEAA_NPEBD@Z
         33   20 00015ED0 ?InitLog@QQIpcChildWrapper@qqipc@qqimpl@@QEAAXHPEAX@Z
         34   21 00015F40 ?InitLog@QQIpcParentWrapper@qqipc@qqimpl@@QEAAXHPEAX@Z
```

因此，我们需要重新编译 QQImpl 项目。

首先，从 [QQImpl](https://github.com/EEEEhex/QQImpl) 克隆项目，比如克隆到 `D:\projects\dotnet` 目录，当前最新的提交为 `3aac297`。解压 *3rdparty* 目录下的 *3rdparty.7z* 文件

{% asset_img dll-3.png %}

解压 *xplugin_protobuf* 目录下的 *google.zip* 文件

{% asset_img dll-4.png %}

从 [CMake](https://cmake.org/) 网站下载 [cmake-3.31.6-windows-x86_64.msi](https://github.com/Kitware/CMake/releases/download/v3.31.6/cmake-3.31.6-windows-x86_64.msi) 进行安装，在终端输入 `cmake -version` 验证安装是否成功。在终端进入 *D:\projects\dotnet\QQImpl\MMMojoCall* 目录，输入如下命令生成解决方案

```shell
cmake -B build -G "Visual Studio 17 2022" -A x64 -DXPLUGIN_WRAPPER=ON -DBUILD_QQIPC=ON -DBUILD_CPPEXAMPLE=ON -DEXAMPLE_USE_JSON=ON -DBUILD_PURE_C_MODE=ON
```

等待命令执行完成后，使用 Visual Studio 打开 *build* 目录，即 *D:\projects\dotnet\QQImpl\MMMojoCall\build*，中的解决方案

{% asset_img dll-5.png %}

在弹出的 *Open Project/Solution* 对话中选择刚生成的解决方案

{% asset_img dll-6.png %}

打开解决方案后在工具栏选择 *Release*

{% asset_img dll-7.png %}

随后在菜单栏选择 *Build > Build Solution* 菜单

{% asset_img dll-8.png %}

编译完成后进入 *D:\projects\dotnet\QQImpl\MMMojoCall\build\Release* 目录，用编译好的 *MMMojoCall.dll* 替换 *QQScreenShotLauncher* 中的同名文件

{% asset_img dll-9.png %}

我们也可以使用 `dumpbin /EXPORTS .\MMMojoCall.dll` 命令检查是否有 *qq_mojoipc* 相关的内容

```text
Microsoft (R) COFF/PE Dumper Version 14.41.34120.0
Copyright (C) Microsoft Corporation.  All rights reserved.


Dump of file .\MMMojoCall.dll

File Type: DLL

  Section contains the following exports for MMMojoCall.dll

    00000000 characteristics
    FFFFFFFF time date stamp
        0.00 version
           1 ordinal base
         105 number of functions
         105 number of names

    ordinal hint RVA      name

          1    0 00005CE0 ??0OCRManager@mmmojocall@qqimpl@@QEAA@XZ
          2    1 00007580 ??0PlayerManager@mmmojocall@qqimpl@@QEAA@XZ
          3    2 0001AC10 ??0QQIpcChildWrapper@qqipc@qqimpl@@QEAA@AEBV012@@Z
          4    3 0001AC40 ??0QQIpcChildWrapper@qqipc@qqimpl@@QEAA@XZ
          5    4 0001AC10 ??0QQIpcParentWrapper@qqipc@qqimpl@@QEAA@AEBV012@@Z
          6    5 0001AC40 ??0QQIpcParentWrapper@qqipc@qqimpl@@QEAA@XZ
          7    6 0000A1B0 ??0UtilityManager@mmmojocall@qqimpl@@QEAA@XZ
          8    7 00003240 ??0XPluginManager@mmmojocall@qqimpl@@QEAA@AEBV012@@Z
          9    8 00003420 ??0XPluginManager@mmmojocall@qqimpl@@QEAA@XZ
         10    9 00005F60 ??1OCRManager@mmmojocall@qqimpl@@QEAA@XZ
         11    A 000077E0 ??1PlayerManager@mmmojocall@qqimpl@@QEAA@XZ
         12    B 0001ADD0 ??1QQIpcChildWrapper@qqipc@qqimpl@@QEAA@XZ
         13    C 0001ADD0 ??1QQIpcParentWrapper@qqipc@qqimpl@@QEAA@XZ
         14    D 0000A2C0 ??1UtilityManager@mmmojocall@qqimpl@@QEAA@XZ
         15    E 00003720 ??1XPluginManager@mmmojocall@qqimpl@@QEAA@XZ
         16    F 0001AE50 ??4QQIpcChildWrapper@qqipc@qqimpl@@QEAAAEAV012@AEBV012@@Z
         17   10 0001AE50 ??4QQIpcParentWrapper@qqipc@qqimpl@@QEAAAEAV012@AEBV012@@Z
         18   11 00003880 ??4XPluginManager@mmmojocall@qqimpl@@QEAAAEAV012@AEBV012@@Z
         19   12 00003AD0 ?AppendSwitchNativeCmdLine@XPluginManager@mmmojocall@qqimpl@@QEAA_NPEBD0@Z
         20   13 00006190 ?CallUsrCallback@OCRManager@mmmojocall@qqimpl@@QEAAXHPEBXH@Z
         21   14 0000A360 ?CallUsrCallback@UtilityManager@mmmojocall@qqimpl@@QEAAXHPEBXHV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@@Z
         22   15 0001AEA0 ?ConnectedToChildProcess@QQIpcParentWrapper@qqipc@qqimpl@@QEAA_NH@Z
         23   16 00007AB0 ?CreateCoreStatusSync@PlayerManager@mmmojocall@qqimpl@@AEAAXH@Z
         24   17 00007B70 ?CreatePlayerCore@PlayerManager@mmmojocall@qqimpl@@QEAAHXZ
         25   18 00007E20 ?DeleteCoreStatusSync@PlayerManager@mmmojocall@qqimpl@@AEAAXH@Z
         26   19 00007FC0 ?DestroyPlayerCore@PlayerManager@mmmojocall@qqimpl@@QEAAXH@Z
         27   1A 000064A0 ?DoOCRTask@OCRManager@mmmojocall@qqimpl@@QEAA_NPEBD@Z
         28   1B 0000A6E0 ?DoPicQRScan@UtilityManager@mmmojocall@qqimpl@@QEAA_NPEBDH@Z
         29   1C 0000AAD0 ?DoResampleImage@UtilityManager@mmmojocall@qqimpl@@QEAA_NV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@0HH@Z
         30   1D 00006820 ?GetConnectState@OCRManager@mmmojocall@qqimpl@@QEAA_NXZ
         31   1E 00008130 ?GetConnectState@PlayerManager@mmmojocall@qqimpl@@QEAA_NXZ
         32   1F 0000ADF0 ?GetConnectState@UtilityManager@mmmojocall@qqimpl@@QEAA_NXZ
         33   20 00008140 ?GetCurrentPosition@PlayerManager@mmmojocall@qqimpl@@QEAAHH@Z
         34   21 000081D0 ?GetIdlePlayerCoreId@PlayerManager@mmmojocall@qqimpl@@AEAAHXZ
         35   22 00006830 ?GetIdleTaskId@OCRManager@mmmojocall@qqimpl@@AEAAHXZ
         36   23 0001AF70 ?GetLastErrStr@QQIpcChildWrapper@qqipc@qqimpl@@QEAAPEBDXZ
         37   24 0001AF70 ?GetLastErrStr@QQIpcParentWrapper@qqipc@qqimpl@@QEAAPEBDXZ
         38   25 00003CD0 ?GetLastErrStr@XPluginManager@mmmojocall@qqimpl@@QEAAPEBDXZ
         39   26 00008280 ?GetPlayerCoreNum@PlayerManager@mmmojocall@qqimpl@@QEAAHXZ
         40   27 00008290 ?GetPlayerCoreStatus@PlayerManager@mmmojocall@qqimpl@@QEAAPEAUCoreStatus@123@H@Z
         41   28 0001AF80 ?InitChildIpc@QQIpcChildWrapper@qqipc@qqimpl@@QEAAXXZ
         42   29 0001AFE0 ?InitEnv@QQIpcChildWrapper@qqipc@qqimpl@@QEAA_NPEBD@Z
         43   2A 0001B180 ?InitEnv@QQIpcParentWrapper@qqipc@qqimpl@@QEAA_NPEBD@Z
         44   2B 0001B320 ?InitLog@QQIpcChildWrapper@qqipc@qqimpl@@QEAAXHPEAX@Z
         45   2C 0001B390 ?InitLog@QQIpcParentWrapper@qqipc@qqimpl@@QEAAXHPEAX@Z
         46   2D 0001B410 ?InitParentIpc@QQIpcParentWrapper@qqipc@qqimpl@@QEAAXXZ
         47   2E 000082F0 ?InitPlayerCore@PlayerManager@mmmojocall@qqimpl@@QEAAHHPEAUHWND__@@PEBD1_J_N3M3M@Z
         48   2F 000068E0 ?KillWeChatOCR@OCRManager@mmmojocall@qqimpl@@QEAAXXZ
         49   30 00008640 ?KillWeChatPlayer@PlayerManager@mmmojocall@qqimpl@@QEAAXXZ
         50   31 0000AE00 ?KillWeChatUtility@UtilityManager@mmmojocall@qqimpl@@QEAAXXZ
         51   32 0001B470 ?LaunchChildProcess@QQIpcParentWrapper@qqipc@qqimpl@@QEAAHPEBDP6AXPEAXPEADH2H@Z1PEAPEADH@Z
         52   33 00008650 ?MuteVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NH_N@Z
         53   34 0001B620 ?OnDefaultReceiveMsg@QQIpcParentWrapper@qqipc@qqimpl@@SAXPEAXPEADH1H@Z
         54   35 000087C0 ?PauseVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NH@Z
         55   36 0001B730 ?ReLaunchChildProcess@QQIpcParentWrapper@qqipc@qqimpl@@QEAAHH@Z
         56   37 00008E80 ?RepeatVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NH_N@Z
         57   38 00008FF0 ?ResumeVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NH@Z
         58   39 00009140 ?ResziePlayerCore@PlayerManager@mmmojocall@qqimpl@@QEAA_NHHH@Z
         59   3A 000092C0 ?RunEvent@PlayerManager@mmmojocall@qqimpl@@QEAAXHH@Z
         60   3B 00009330 ?SeekToVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NHH@Z
         61   3C 0001B7E0 ?SendIpcMessage@QQIpcChildWrapper@qqipc@qqimpl@@QEAAXPEBD0H@Z
         62   3D 0001B860 ?SendIpcMessage@QQIpcParentWrapper@qqipc@qqimpl@@QEAA_NHPEBD0H@Z
         63   3E 00006A10 ?SendOCRTask@OCRManager@mmmojocall@qqimpl@@AEAA_NIV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@@Z
         64   3F 00003CE0 ?SendPbSerializedData@XPluginManager@mmmojocall@qqimpl@@QEAAXPEAXHH_NI@Z
         65   40 00006C90 ?SetCallbackDataMode@OCRManager@mmmojocall@qqimpl@@QEAAX_N@Z
         66   41 0000AE10 ?SetCallbackDataMode@UtilityManager@mmmojocall@qqimpl@@QEAAX_N@Z
         67   42 00003D60 ?SetCallbackUsrData@XPluginManager@mmmojocall@qqimpl@@QEAAXPEAX@Z
         68   43 00003D70 ?SetCallbacks@XPluginManager@mmmojocall@qqimpl@@QEAAXUMMMojoEnvironmentCallbacks@common@mmmojo@@@Z
         69   44 0001B950 ?SetChildReceiveCallback@QQIpcChildWrapper@qqipc@qqimpl@@QEAAXP6AXPEAXPEADH1H@Z@Z
         70   45 00006CA0 ?SetConnectState@OCRManager@mmmojocall@qqimpl@@QEAAX_N@Z
         71   46 00009580 ?SetConnectState@PlayerManager@mmmojocall@qqimpl@@QEAAX_N@Z
         72   47 0000AE20 ?SetConnectState@UtilityManager@mmmojocall@qqimpl@@QEAAX_N@Z
         73   48 00003DA0 ?SetExePath@XPluginManager@mmmojocall@qqimpl@@QEAA_NPEBD@Z
         74   49 00003FE0 ?SetLastErrStr@XPluginManager@mmmojocall@qqimpl@@IEAAXV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@@Z
         75   4A 0001B9D0 ?SetLogLevel@QQIpcParentWrapper@qqipc@qqimpl@@QEAAXH@Z
         76   4B 00004080 ?SetOneCallback@XPluginManager@mmmojocall@qqimpl@@QEAAXHPEAX@Z
         77   4C 00009590 ?SetPlayerCoreIdIdle@PlayerManager@mmmojocall@qqimpl@@AEAA_NH@Z
         78   4D 0000AEA0 ?SetReadOnPull@UtilityManager@mmmojocall@qqimpl@@QEAAXP6AXHPEBXH@Z@Z
         79   4E 00006D20 ?SetReadOnPush@OCRManager@mmmojocall@qqimpl@@QEAAXP6AXPEBDPEBXH@Z@Z
         80   4F 0000AEB0 ?SetReadOnPush@UtilityManager@mmmojocall@qqimpl@@QEAAXP6AXHPEBXH@Z@Z
         81   50 000095B0 ?SetSpeedVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NHM@Z
         82   51 00009730 ?SetSurfaceVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NHPEAUHWND__@@@Z
         83   52 000098B0 ?SetSwitchArgs@PlayerManager@mmmojocall@qqimpl@@QEAA_NPEBD00@Z
         84   53 00006D30 ?SetTaskIdIdle@OCRManager@mmmojocall@qqimpl@@AEAA_NH@Z
         85   54 00006D50 ?SetUsrLibDir@OCRManager@mmmojocall@qqimpl@@QEAA_NPEBD@Z
         86   55 0000AEC0 ?SetUsrLibDir@UtilityManager@mmmojocall@qqimpl@@QEAA_NPEBD@Z
         87   56 00009920 ?SetVolumeVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NHM@Z
         88   57 00004100 ?StartMMMojoEnv@XPluginManager@mmmojocall@qqimpl@@QEAA_NXZ
         89   58 00009AA0 ?StartVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NH@Z
         90   59 00006DB0 ?StartWeChatOCR@OCRManager@mmmojocall@qqimpl@@QEAA_NXZ
         91   5A 00009C00 ?StartWeChatPlayer@PlayerManager@mmmojocall@qqimpl@@QEAA_NXZ
         92   5B 0000AF20 ?StartWeChatUtility@UtilityManager@mmmojocall@qqimpl@@QEAA_NXZ
         93   5C 00004830 ?StopMMMojoEnv@XPluginManager@mmmojocall@qqimpl@@QEAAXXZ
         94   5D 00009C30 ?StopVideo@PlayerManager@mmmojocall@qqimpl@@QEAA_NH@Z
         95   5E 0001BA40 ?TerminateChildProcess@QQIpcParentWrapper@qqipc@qqimpl@@QEAA_NHH_N@Z
         96   5F 00009D90 ?WaitEvent@PlayerManager@mmmojocall@qqimpl@@QEAAXHH@Z
         97   60 000054C0 CallFuncXPluginMgr
         98   61 00005B70 GetInstanceXPluginMgr
         99   62 00004C70 GetPbSerializedData
        100   63 00004CB0 GetReadInfoAttachData
        101   64 00004CF0 InitMMMojoDLLFuncs
        102   65 00005450 InitMMMojoGlobal
        103   66 00005C10 ReleaseInstanceXPluginMgr
        104   67 00005480 RemoveReadInfo
        105   68 00005490 ShutdownMMMojoGlobal

  Summary

        8000 .data
        C000 .pdata
       38000 .rdata
        3000 .reloc
        1000 .rsrc
       FB000 .text
```

## 包装 QQImpl 动态链接库

### 创建动态链接库

在 *Solution Explorer* 选中 *QQScreenShotLauncher* 解决方案，在右键菜单中选择 *Add > New Project*

{% asset_img create-project-1.png %}

在 *Add a new project* 对话框选择 *C++*、*Windows*、*Library*、*Dynamic-Link Library (DLL)*

{% asset_img wrapper-1.png %}

在 *Configure your new project* 对话框填写 *Project name* 为 *MMMojoCallWrapper*，*Location* 保持默认

{% asset_img wrapper-2.png %}

### 添加 .h 和 .lib 文件

在 *QQImpl* 项目找到 *qq_ipc.h* 文件，把它复制到 *D:\projects\dotnet\QQScreenShotLauncher\MMMojoCallWrapper* 目录

{% asset_img wrapper-3.png %}

在 *QQImpl* 项目找到 *MMMojoCall.lib* 文件，把它复制到 *D:\projects\dotnet\QQScreenShotLauncher\MMMojoCallWrapper* 目录

{% asset_img wrapper-4.png %}

在 *Solution Explorer* 选中 *MMMojoCallWrapper* 项目的 *Header Files*，在右键菜单选择 *Add > Existing Item*

{% asset_img wrapper-5.png %}

在 *Add Existing Item* 对话框选择刚刚添加的 *qq_ipc.h* 文件

{% asset_img wrapper-6.png %}

对 *qq_ipc.h* 头文件依赖的 *mojo_call_export.h* 头文件进行相同的操作。

### 添加动态链接库包装

在 *Solution Explorer* 选中 *MMMojoCallWrapper* 项目，在右键菜单中选择 *Add > Class*

{% asset_img wrapper-7.png %}

在 *Add Class* 对话框输入类名 *MMMojoCallWrapper*，其他保持默认

{% asset_img wrapper-8.png %}

下面的内容参考以下资料

1. [C#与C++交互开发系列（十一）：委托和函数指针传递](https://blog.csdn.net/houbincarson/article/details/143191834)
2. [将委托作为回调方法进行封送](https://learn.microsoft.com/zh-cn/dotnet/framework/interop/marshalling-a-delegate-as-a-callback-method)

*MMMojoCallWrapper.h* 文件的内容如下所示

```c++
#pragma once

#ifdef MMMOJOCALLWRAPPER_EXPORTS
#define MMMOJOCALLWRAPPER_API __declspec(dllexport)
#else
#define MMMOJOCALLWRAPPER_API __declspec(dllimport)
#endif

extern "C"  MMMOJOCALLWRAPPER_API typedef void (*CallbackIpc)(void*, char*, int, char*, int);

// QQIpcParentWrapper

extern "C" MMMOJOCALLWRAPPER_API void* CreateQQIpcParentWrapper();

extern "C" MMMOJOCALLWRAPPER_API void DeleteQQIpcParentWrapper(void* instance);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_OnDefaultReceiveMsg(void* pArg, char* msg, int arg3, char* addition_msg, int addition_msg_size);

extern "C" MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_InitEnv(void* instance, const char* dll_path);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_SetLogLevel(void* instance, int level);

extern "C" MMMOJOCALLWRAPPER_API const char* QQIpcParentWrapper_GetLastErrStr(void* instance);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_InitLog(void* instance, int level, void* callback);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_InitParentIpc(void* instance);

extern "C" MMMOJOCALLWRAPPER_API int QQIpcParentWrapper_LaunchChildProcess(void* instance, const char* file_path, CallbackIpc callback, void* cb_arg, char** cmdlines, int cmd_num);

extern "C" MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_ConnectedToChildProcess(void* instance, int pid);

extern "C" MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_SendIpcMessage(void* instance, int pid, const char* command, const char* addition_msg, int addition_msg_size);

extern "C" MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_TerminateChildProcess(void* instance, int pid, int exit_code, bool wait_);

extern "C" MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_ReLaunchChildProcess(void* instance, int pid);

// QQIpcChildWrapper

extern "C" MMMOJOCALLWRAPPER_API void* CreateQQIpcChildWrapper();

extern "C" MMMOJOCALLWRAPPER_API void DeleteQQIpcChildWrapper(void* instance);

extern "C" MMMOJOCALLWRAPPER_API const char* QQIpcChildWrapper_GetLastErrStr(void* instance);

extern "C" MMMOJOCALLWRAPPER_API bool QQIpcChildWrapper_InitEnv(void* instance, const char* dll_path);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_InitChildIpc(void* instance);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_InitLog(void* instance, int level, void* callback);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_SetChildReceiveCallback(void* instance, CallbackIpc callback);

extern "C" MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_SendIpcMessage(void* instance, const char* command, const char* addition_msg, int addition_msg_size);
```

*MMMojoCallWrapper.cpp* 文件的内容如下所示

```c++
#include "pch.h"
#include "MMMojoCallWrapper.h"
#include "qq_ipc.h"

#pragma comment(lib, "MMMojoCall.lib")

// QQIpcParentWrapper

MMMOJOCALLWRAPPER_API void* CreateQQIpcParentWrapper()
{
	return new qqimpl::qqipc::QQIpcParentWrapper();
}

MMMOJOCALLWRAPPER_API void DeleteQQIpcParentWrapper(void* instance)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	delete o;
}

MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_OnDefaultReceiveMsg(void* pArg, char* msg, int arg3, char* addition_msg, int addition_msg_size)
{
	return qqimpl::qqipc::QQIpcParentWrapper::OnDefaultReceiveMsg(pArg, msg, arg3, addition_msg, addition_msg_size);
}

MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_InitEnv(void* instance, const char* dll_path)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->InitEnv(dll_path);
}

MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_SetLogLevel(void* instance, int level)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->SetLogLevel(level);
}

MMMOJOCALLWRAPPER_API const char* QQIpcParentWrapper_GetLastErrStr(void* instance)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->GetLastErrStr();
}

MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_InitLog(void* instance, int level, void* callback)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->InitLog(level, callback);
}

MMMOJOCALLWRAPPER_API void QQIpcParentWrapper_InitParentIpc(void* instance)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->InitParentIpc();
}

MMMOJOCALLWRAPPER_API int QQIpcParentWrapper_LaunchChildProcess(void* instance, const char* file_path, CallbackIpc callback, void* cb_arg, char** cmdlines, int cmd_num)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->LaunchChildProcess(file_path, callback, cb_arg, cmdlines, cmd_num);
}

MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_ConnectedToChildProcess(void* instance, int pid)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->ConnectedToChildProcess(pid);
}

MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_SendIpcMessage(void* instance, int pid, const char* command, const char* addition_msg, int addition_msg_size)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->SendIpcMessage(pid, command, addition_msg, addition_msg_size);
}

MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_TerminateChildProcess(void* instance, int pid, int exit_code, bool wait_)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->TerminateChildProcess(pid, exit_code, wait_);
}

MMMOJOCALLWRAPPER_API bool QQIpcParentWrapper_ReLaunchChildProcess(void* instance, int pid)
{
	qqimpl::qqipc::QQIpcParentWrapper* o = static_cast<qqimpl::qqipc::QQIpcParentWrapper*>(instance);
	return o->ReLaunchChildProcess(pid);
}

// QQIpcChildWrapper

MMMOJOCALLWRAPPER_API void* CreateQQIpcChildWrapper()
{
	return new qqimpl::qqipc::QQIpcChildWrapper();
}

MMMOJOCALLWRAPPER_API void DeleteQQIpcChildWrapper(void* instance)
{
	qqimpl::qqipc::QQIpcChildWrapper* o = static_cast<qqimpl::qqipc::QQIpcChildWrapper*>(instance);
	delete o;
}

MMMOJOCALLWRAPPER_API const char* QQIpcChildWrapper_GetLastErrStr(void* instance)
{
	qqimpl::qqipc::QQIpcChildWrapper* o = static_cast<qqimpl::qqipc::QQIpcChildWrapper*>(instance);
	return o->GetLastErrStr();
}

MMMOJOCALLWRAPPER_API bool QQIpcChildWrapper_InitEnv(void* instance, const char* dll_path)
{
	qqimpl::qqipc::QQIpcChildWrapper* o = static_cast<qqimpl::qqipc::QQIpcChildWrapper*>(instance);
	return o->InitEnv(dll_path);
}

MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_InitChildIpc(void* instance)
{
	qqimpl::qqipc::QQIpcChildWrapper* o = static_cast<qqimpl::qqipc::QQIpcChildWrapper*>(instance);
	return o->InitChildIpc();
}

MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_InitLog(void* instance, int level, void* callback)
{
	qqimpl::qqipc::QQIpcChildWrapper* o = static_cast<qqimpl::qqipc::QQIpcChildWrapper*>(instance);
	return o->InitLog(level, callback);
}

MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_SetChildReceiveCallback(void* instance, CallbackIpc callback)
{
	qqimpl::qqipc::QQIpcChildWrapper* o = static_cast<qqimpl::qqipc::QQIpcChildWrapper*>(instance);
	return o->SetChildReceiveCallback(callback);
}

MMMOJOCALLWRAPPER_API void QQIpcChildWrapper_SendIpcMessage(void* instance, const char* command, const char* addition_msg, int addition_msg_size)
{
	qqimpl::qqipc::QQIpcChildWrapper* o = static_cast<qqimpl::qqipc::QQIpcChildWrapper*>(instance);
	return o->SendIpcMessage(command, addition_msg, addition_msg_size);
}
```

编译 *MMMojoCallWrapper* 项目，在终端进入 *D:\projects\dotnet\QQScreenShotLauncher\x64\Debug* 目录执行 `dumpbin /exports .\MMMojoCallWrapper.dll` 命令

```text
Microsoft (R) COFF/PE Dumper Version 14.41.34120.0
Copyright (C) Microsoft Corporation.  All rights reserved.


Dump of file .\MMMojoCallWrapper.dll

File Type: DLL

  Section contains the following exports for MMMojoCallWrapper.dll

    00000000 characteristics
    FFFFFFFF time date stamp
        0.00 version
           1 ordinal base
          21 number of functions
          21 number of names

    ordinal hint RVA      name

          1    0 00011424 CreateQQIpcChildWrapper = @ILT+1055(CreateQQIpcChildWrapper)
          2    1 000110E1 CreateQQIpcParentWrapper = @ILT+220(CreateQQIpcParentWrapper)
          3    2 000112F8 DeleteQQIpcChildWrapper = @ILT+755(DeleteQQIpcChildWrapper)
          4    3 00011375 DeleteQQIpcParentWrapper = @ILT+880(DeleteQQIpcParentWrapper)
          5    4 000112DA QQIpcChildWrapper_GetLastErrStr = @ILT+725(QQIpcChildWrapper_GetLastErrStr)
          6    5 00011460 QQIpcChildWrapper_InitChildIpc = @ILT+1115(QQIpcChildWrapper_InitChildIpc)
          7    6 00011244 QQIpcChildWrapper_InitEnv = @ILT+575(QQIpcChildWrapper_InitEnv)
          8    7 000111A4 QQIpcChildWrapper_InitLog = @ILT+415(QQIpcChildWrapper_InitLog)
          9    8 0001146F QQIpcChildWrapper_SendIpcMessage = @ILT+1130(QQIpcChildWrapper_SendIpcMessage)
         10    9 00011014 QQIpcChildWrapper_SetChildReceiveCallback = @ILT+15(QQIpcChildWrapper_SetChildReceiveCallback)
         11    A 000112D5 QQIpcParentWrapper_ConnectedToChildProcess = @ILT+720(QQIpcParentWrapper_ConnectedToChildProcess)
         12    B 000112C6 QQIpcParentWrapper_GetLastErrStr = @ILT+705(QQIpcParentWrapper_GetLastErrStr)
         13    C 000113BB QQIpcParentWrapper_InitEnv = @ILT+950(QQIpcParentWrapper_InitEnv)
         14    D 0001112C QQIpcParentWrapper_InitLog = @ILT+295(QQIpcParentWrapper_InitLog)
         15    E 000112B2 QQIpcParentWrapper_InitParentIpc = @ILT+685(QQIpcParentWrapper_InitParentIpc)
         16    F 00011113 QQIpcParentWrapper_LaunchChildProcess = @ILT+270(QQIpcParentWrapper_LaunchChildProcess)
         17   10 0001143D QQIpcParentWrapper_OnDefaultReceiveMsg = @ILT+1080(QQIpcParentWrapper_OnDefaultReceiveMsg)
         18   11 00011302 QQIpcParentWrapper_ReLaunchChildProcess = @ILT+765(QQIpcParentWrapper_ReLaunchChildProcess)
         19   12 000113D4 QQIpcParentWrapper_SendIpcMessage = @ILT+975(QQIpcParentWrapper_SendIpcMessage)
         20   13 000110BE QQIpcParentWrapper_SetLogLevel = @ILT+185(QQIpcParentWrapper_SetLogLevel)
         21   14 000110F0 QQIpcParentWrapper_TerminateChildProcess = @ILT+235(QQIpcParentWrapper_TerminateChildProcess)

  Summary

        1000 .00cfg
        1000 .data
        2000 .idata
        1000 .msvcjmc
        3000 .pdata
        4000 .rdata
        1000 .reloc
        1000 .rsrc
        A000 .text
       10000 .textbss
```

## 实现 QQ 截图功能

这部分内容参考以下资料

1. [使用非托管 DLL 函数](https://learn.microsoft.com/zh-cn/dotnet/framework/interop/consuming-unmanaged-dll-functions)
2. [用vs完整的搭建一个项目流程(包括多个项目之间的依赖) 方法一](https://blog.csdn.net/qq_38409301/article/details/121869272)
3. [Visual Studio 2019：引用动态DLL项目](https://blog.csdn.net/zhizhengguan/article/details/112764805)

### 添加项目依赖

在 *Solution Explorer* 选中 *NTLauncher* 项目，在右键菜单中选择 *Build Dependencies > Project Dependencies*

{% asset_img shot-1.png %}

在弹出的 *Project Dependencies* 对话框中选中 *MMMojoCallWrapper*

{% asset_img shot-2.png %}

选中 *NTLauncher* 项目，在右键菜单中选择 *Properties*

{% asset_img add-icon-1.png %}

选中 *Build > Events*，在 *Post-build event* 输入 `xcopy /y /d "$(SolutionDir)x64\$(Configuration)\$(IntDir)MMMojoCallWrapper.dll" "$(OutDir)"`

{% asset_img shot-3.png %}

### 实现动态链接库包装

在 *Solution Explorer* 选中 *NTLauncher* 项目，在右键菜单中选择 *Add Class*

{% asset_img shot-4.png %}

在弹出的 *Add New Item* 对话框中创建一个 *Class*，名称为 *QQIpcWrapper.cs*

{% asset_img shot-5.png %}
