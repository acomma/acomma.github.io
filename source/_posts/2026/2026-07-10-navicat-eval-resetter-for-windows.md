---
title: 重置 Windows 版 Navicat 的试用时间
date: 2026-07-10 23:50:05
updated: 2026-07-10 23:50:05
tags: [Navicat]
---

## 现有脚本的问题

网上有很多重置 Windows 版 Navicat 使用时间的脚本，重置的方式是在 Windows 注册表中删除以下键

* `HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\Registration17XEN`，具体的键视安装的版本而定
* `HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\Update`
* `HKEY_CURRENT_USER\Software\Classes\CLSID\{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}`，这个键下面包含子健 `Info`
* `HKEY_CURRENT_USER\Software\Classes\CLSID\{YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY}`，这个键下面包含子健 `ShellInfo`

不同的安装 `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` 和 `{YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY}` 不一样。哪些脚本对前两项的处理问题不大，但对后两项的处理比较粗糙，存在误删问题，比如可能误删 *OneDrive* 的注册表键

{% asset_img onedrive_clsid.png %}

此事在[谨慎使用网上流传的Navicat重置试用期bat脚本](https://zhuanlan.zhihu.com/p/689194655)一文中亦有记载。

<!-- more -->

## 方法一：限制子健数量和名称

通过对比分析注册表在重置前后的差异发现 `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` 只会有一个子健 `Info`，`{YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY}` 只会有两个子健 `ShellInfo` 和 `DefaultIcon` 且同时出现，举个例子

{% asset_img candidate_clsid.png %}

因此增强重置脚本的方式是限制子健的数量和名称，下面是一个 Python 脚本

```python
import winreg


def get_sub_key_names(key: winreg.HKEYType) -> list[str]:
    sub_key_names: list[str] = []
    index = 0

    while True:
        try:
            sub_key_name = winreg.EnumKey(key, index)
            sub_key_names.append(sub_key_name)
            index = index + 1
        except OSError:
            break

    return sub_key_names


def get_candidate_class_ids() -> tuple[list[str], list[str]]:
    root_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID")
    class_ids = get_sub_key_names(root_key)
    winreg.CloseKey(root_key)

    infos = []
    shell_folders = []

    for class_id in class_ids:
        if class_id.startswith("{CAFEEFAC-"):
            continue

        class_id_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + class_id)
        sub_key_names = get_sub_key_names(class_id_key)
        winreg.CloseKey(class_id_key)

        if len(sub_key_names) == 1:
            if sub_key_names[0] == "Info":
                infos.append(class_id)

        if len(sub_key_names) == 2:
            has_default_icon = False
            has_shell_folder = False
            for sub_key_name in sub_key_names:
                if sub_key_name == "DefaultIcon":
                    has_default_icon = True
                if sub_key_name == "ShellFolder":
                    has_shell_folder = True
            if has_default_icon and has_shell_folder:
                shell_folders.append(class_id)

    return infos, shell_folders


def show_candidate_class_ids():
    infos, shell_folders = get_candidate_class_ids()

    print("包含 Info 的键：")
    for class_id in infos:
        print(f"    {class_id}")

    print("包含 DefaultIcon 和 ShellFolder 的键：")
    for class_id in shell_folders:
        print(f"    {class_id}")


def remove_class_ids():
    infos, shell_folders = get_candidate_class_ids()

    for class_id in infos:
        print(rf"删除包含 Info 的键：HKEY_CURRENT_USER\Software\Classes\CLSID\{class_id}")
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + class_id + "\\" + "Info")
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + class_id)

    for class_id in shell_folders:
        print(rf"删除包含 DefaultIcon 和 ShellFolder 的键：HKEY_CURRENT_USER\Software\Classes\CLSID\{class_id}")
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + class_id + "\\" + "DefaultIcon")
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + class_id + "\\" + "ShellFolder")
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + class_id)


def remove_registration():
    premium_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\PremiumSoft\NavicatPremium")
    sub_key_names = get_sub_key_names(premium_key)
    winreg.CloseKey(premium_key)

    for sub_key_name in sub_key_names:
        if sub_key_name == "Registration":
            continue
        if not sub_key_name.startswith("Registration"):
            continue
        print(rf"删除 Registration 键：HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\{sub_key_name}")
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\PremiumSoft\NavicatPremium" + "\\" + sub_key_name)

    print(rf"删除 Update 键：HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\Update")
    winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\PremiumSoft\NavicatPremium\Update")

if __name__ == "__main__":
    remove_registration()
    remove_class_ids()
```

## 方法二：使用进程监视器精确定位

上面的增强方式还是有缺陷，因为其他软件可能也会有相同的键名称使用方式。一种改进的方法是通过 [Sysinternals](https://learn.microsoft.com/zh-cn/sysinternals/) 提供的[进程监视器](https://learn.microsoft.com/zh-cn/sysinternals/downloads/procmon)找到目标键，为了实现目标需要配置过滤器

{% asset_img procmon_filter.png %}

配置好过滤器后重启 Navicat，然后就可以看到类似下面的结果

{% asset_img procmon.png %}

从上图中就可以发现 Navicat 使用的包含 `Info` 的键为 `{A3B39FF1-93A2-0A91-4EF5-B5F4E6820247}`，这个键对同一台电脑上的同一个版本的 Navicat 是不变的。

通过*进程监视器*没能找到包含 `ShellFolder` 的键，但我们观察到 `Info` 键的值名称 `0F6CEEEDC2FAF4117BD48440C3629FC1` 的前 24 位，即 `0F6CEEEDC2FAF4117BD48440`，与包含 `ShellFolder` 键的 `{0F6CEEED-C2FA-F411-7BD4-8440C3629FC1}` 的前面部分，即 `{0F6CEEED-C2FA-F411-7BD4-8440`，一样。这个发现在另一台电脑上得到了验证

{% asset_img prefix1.png %}

{% asset_img prefix2.png %}

同时在同一台电脑上观察多次删除注册表并重启 Navicat 后注册表的结果发现 `{YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY}` 的具体值每次都不一样，但是它前面部分与 `Info` 键的值名称的前面部分一样。

基于以上发现可以实现一个更精准的删除注册表键的 Python 脚本

```python
import winreg
from typing import Any


def get_sub_key_names(key: winreg.HKEYType) -> list[str]:
    sub_key_names: list[str] = []
    index = 0

    while True:
        try:
            sub_key_name = winreg.EnumKey(key, index)
            sub_key_names.append(sub_key_name)
            index = index + 1
        except OSError:
            break

    return sub_key_names


def get_values(key: winreg.HKEYType) -> list[tuple[str, Any, int]]:
    values: list[tuple[str, Any, int]] = []
    index = 0

    while True:
        try:
            value_name, value_data, data_type = winreg.EnumValue(key, index)
            values.append((value_name, value_data, data_type))
            index = index + 1
        except OSError:
            break

    return values


def remove_registration():
    premium_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\PremiumSoft\NavicatPremium")
    sub_key_names = get_sub_key_names(premium_key)
    winreg.CloseKey(premium_key)

    for sub_key_name in sub_key_names:
        if sub_key_name == "Registration":
            continue
        if not sub_key_name.startswith("Registration"):
            continue
        print(rf"删除 Registration 键：HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\{sub_key_name}")
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\PremiumSoft\NavicatPremium" + "\\" + sub_key_name)

    print(rf"删除 Update 键：HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\Update")
    winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\PremiumSoft\NavicatPremium\Update")

def remove_class_ids():
    # 通过 https://learn.microsoft.com/zh-cn/sysinternals/downloads/procmon 程序找到要删除 Info 对应的 CLSID，这是唯一需要替换的地方
    info_class_id = "{BE501491-0113-1CCF-D535-F9E9E0FF7462}"

    info_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + info_class_id + "\\" + "Info")
    values = get_values(info_key)
    winreg.CloseKey(info_key)

    if len(values) == 0 or len(values) > 1:
        print("Info 键的值项数量不为 1，暂停删除 CLSID")
        return

    value_name, value_data, data_type = values[0]
    shell_folder_class_id_prefix = "{" + f"{value_name[:8]}-{value_name[8:12]}-{value_name[12:16]}-{value_name[16:20]}-{value_name[20:24]}"

    root_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID")
    sub_key_names = get_sub_key_names(root_key)
    winreg.CloseKey(root_key)

    shell_folder_class_ids = []

    for sub_key_name in sub_key_names:
        if sub_key_name.startswith(shell_folder_class_id_prefix):
            shell_folder_class_ids.append(sub_key_name)

    if len(shell_folder_class_ids) == 0 or len(shell_folder_class_ids) > 1:
        print("ShellFolder 键的数量不为 1，暂停删除 CLSID")
        return

    shell_folder_class_id = shell_folder_class_ids[0]

    root_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + info_class_id)
    sub_key_names = get_sub_key_names(root_key)
    winreg.CloseKey(root_key)

    for sub_key_name in sub_key_names:
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + info_class_id + "\\" + sub_key_name)
    print(rf"删除包含 Info 的键：HKEY_CURRENT_USER\Software\Classes\CLSID\{info_class_id}")
    winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + info_class_id)

    root_key = winreg.OpenKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + shell_folder_class_id)
    sub_key_names = get_sub_key_names(root_key)
    winreg.CloseKey(root_key)

    for sub_key_name in sub_key_names:
        winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + shell_folder_class_id + "\\" + sub_key_name)
    print(rf"删除包含 ShellFolder 的键：HKEY_CURRENT_USER\Software\Classes\CLSID\{shell_folder_class_id}")
    winreg.DeleteKey(winreg.HKEY_CURRENT_USER, r"Software\Classes\CLSID" + "\\" + shell_folder_class_id)


if __name__ == "__main__":
    remove_registration()
    remove_class_ids()
```
