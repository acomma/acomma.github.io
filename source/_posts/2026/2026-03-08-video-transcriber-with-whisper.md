---
title: 使用 Whisper 实现语音转文字
date: 2026-03-08 22:35:43
updated: 2026-03-08 22:35:43
tags: [Python, AI]
---

最近遇到这样一个需求：从视频中提取音频并将语音转换成文字。在和 [Qwen](https://chat.qwen.ai/) 进行一些讨论后决定使用 [openai-whisper](https://pypi.org/project/openai-whisper/) 来实现。

<!-- more -->

## 安装 FFmpeg

*openai-whisper* 依赖 [FFmpeg](https://ffmpeg.org/)，我们首先需要安装它。又因为我是 Windows 系统，因此从 [FFmpeg 下载页](https://ffmpeg.org/download.html) 导航到 [gyan.dev](https://www.gyan.dev/ffmpeg/builds/) 下载 [ffmpeg-release-essentials.zip](https://www.gyan.dev/ffmpeg/builds/ffmpeg-release-essentials.zip)，解压到 *D:\Program Files* 目录，得到 *D:\Program Files\ffmpeg-8.0.1-essentials_build*，配置环境变量 `FFMPEG_HOME=D:\Program Files\ffmpeg-8.0.1-essentials_build`，在 `Path` 变量中加入 `%FFMPEG_HOME%\bin`，打开命令提示符输入 `ffmpeg -version` 检查安装是否成功，当前输出为

```
ffmpeg version 8.0.1-essentials_build-www.gyan.dev Copyright (c) 2000-2025 the FFmpeg developers
built with gcc 15.2.0 (Rev8, Built by MSYS2 project)
configuration: --enable-gpl --enable-version3 --enable-static --disable-w32threads --disable-autodetect --enable-fontconfig --enable-iconv --enable-gnutls --enable-libxml2 --enable-gmp --enable-bzlib --enable-lzma --enable-zlib --enable-libsrt --enable-libssh --enable-libzmq --enable-avisynth --enable-sdl2 --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxvid --enable-libaom --enable-libopenjpeg --enable-libvpx --enable-mediafoundation --enable-libass --enable-libfreetype --enable-libfribidi --enable-libharfbuzz --enable-libvidstab --enable-libvmaf --enable-libzimg --enable-amf --enable-cuda-llvm --enable-cuvid --enable-dxva2 --enable-d3d11va --enable-d3d12va --enable-ffnvcodec --enable-libvpl --enable-nvdec --enable-nvenc --enable-vaapi --enable-openal --enable-libgme --enable-libopenmpt --enable-libopencore-amrwb --enable-libmp3lame --enable-libtheora --enable-libvo-amrwbenc --enable-libgsm --enable-libopencore-amrnb --enable-libopus --enable-libspeex --enable-libvorbis --enable-librubberband
libavutil      60.  8.100 / 60.  8.100
libavcodec     62. 11.100 / 62. 11.100
libavformat    62.  3.100 / 62.  3.100
libavdevice    62.  1.100 / 62.  1.100
libavfilter    11.  4.100 / 11.  4.100
libswscale      9.  1.100 /  9.  1.100
libswresample   6.  1.100 /  6.  1.100

Exiting with exit code 0
```

到此为止，安装 FFmpeg 就完成了。

## 实现脚本

下面是 Vibe Coding 实现的转换脚本 *video_transcriber.py*，项目使用 `uv` 进行管理，需要安装依赖 `uv add openai-whisper==20250625`

```python
import whisper
import argparse
import sys
from pathlib import Path
from datetime import datetime


def transcribe_video(video_path: str, model_size: str, language: str | None, verbose: bool) -> dict:
    """转录视频文件"""
    print(f"🔄 正在加载模型：{model_size} ...")
    model = whisper.load_model(model_size)
    print("✅ 模型加载完成")

    print(f"\n🎬 开始处理：{Path(video_path).name}")
    print(f"   语言：{language if language else '自动检测'}")

    result = model.transcribe(str(video_path), language=language, verbose=verbose)

    print("✅ 转录完成")
    return result


def save_txt(result: dict, output_path: str, with_timestamp: bool) -> str:
    """保存为文件"""
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    if with_timestamp:
        output_file = f"{output_path}_timestamp_{timestamp}.txt"
    else:
        output_file = f"{output_path}_{timestamp}.txt"

    with open(output_file, 'w', encoding='utf-8') as f:
        if with_timestamp:
            for segment in result["segments"]:
                start = segment["start"]
                end = segment["end"]
                text = segment["text"].strip()
                f.write(f"[{start:6.2f}s - {end:6.2f}s] {text}\n")
        else:
            f.write(result["text"])

    print(f"💾 已保存：{output_file}")
    return output_file


def main():
    """主函数"""
    parser = argparse.ArgumentParser(
        description="Whisper 视频转文本工具（简化版）",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
        示例:
            python video_transcriber.py input.mp4
            python video_transcriber.py input.mp4 -m base -l zh
            python video_transcriber.py input.mp4 -o output -t
        """
    )

    parser.add_argument("video", help="视频文件路径")
    parser.add_argument("-m", default="base", choices=["tiny", "base", "small", "medium", "large-v3"], help="模型大小 (默认：base)")
    parser.add_argument("-l", default=None, help="语言代码 (默认：自动检测，中文用 zh)")
    parser.add_argument("-o", default=None, help="输出文件路径 (默认：视频文件名)")
    parser.add_argument("-v", action="store_true", help="显示详细进度")
    parser.add_argument("-t", action="store_true", help="输出包含时间戳")

    args = parser.parse_args()

    # 检查视频文件
    if not Path(args.video).exists():
        print(f"❌ 错误：文件不存在：{args.video}")
        sys.exit(1)

    # 转录
    result = transcribe_video(video_path=args.video, model_size=args.m, language=args.l, verbose=args.v)

    # 保存输出
    output_path = args.o or Path(args.video).stem
    save_txt(result, output_path, args.t)


if __name__ == "__main__":
    main()
```

### 下载 Whisper 模型

在执行 `whisper.load_model()` 这行代码时如果没有找到 `name` 参数指定的模型，会自动下载模型。这是一个耗时的操作，因此我们提前下载好模型。简单的阅读 *whisper* 的源码发现模型的下载路径记录在 [*whisper/\_\_init\_\_.py*](https://github.com/openai/whisper/blob/main/whisper/__init__.py) 文件的 `_MODELS` 字典中

```python
_MODELS = {
    "tiny.en": "https://openaipublic.azureedge.net/main/whisper/models/d3dd57d32accea0b295c96e26691aa14d8822fac7d9d27d5dc00b4ca2826dd03/tiny.en.pt",
    "tiny": "https://openaipublic.azureedge.net/main/whisper/models/65147644a518d12f04e32d6f3b26facc3f8dd46e5390956a9424a650c0ce22b9/tiny.pt",
    "base.en": "https://openaipublic.azureedge.net/main/whisper/models/25a8566e1d0c1e2231d1c762132cd20e0f96a85d16145c3a00adf5d1ac670ead/base.en.pt",
    "base": "https://openaipublic.azureedge.net/main/whisper/models/ed3a0b6b1c0edf879ad9b11b1af5a0e6ab5db9205f891f668f8b0e6c6326e34e/base.pt",
    "small.en": "https://openaipublic.azureedge.net/main/whisper/models/f953ad0fd29cacd07d5a9eda5624af0f6bcf2258be67c92b79389873d91e0872/small.en.pt",
    "small": "https://openaipublic.azureedge.net/main/whisper/models/9ecf779972d90ba49c06d968637d720dd632c55bbf19d441fb42bf17a411e794/small.pt",
    "medium.en": "https://openaipublic.azureedge.net/main/whisper/models/d7440d1dc186f76616474e0ff0b3b6b879abc9d1a4926b7adfa41db2d497ab4f/medium.en.pt",
    "medium": "https://openaipublic.azureedge.net/main/whisper/models/345ae4da62f9b3d59415adc60127b97c714f32e89e936602e85993674d08dcb1/medium.pt",
    "large-v1": "https://openaipublic.azureedge.net/main/whisper/models/e4b87e7e0bf463eb8e6956e646f1e277e901512310def2c24bf0e11bd3c28e9a/large-v1.pt",
    "large-v2": "https://openaipublic.azureedge.net/main/whisper/models/81f7c96c852ee8fc832187b0132e569d6c3065a3252ed18e56effd0b6a73e524/large-v2.pt",
    "large-v3": "https://openaipublic.azureedge.net/main/whisper/models/e5b1a55b89c1367dacf97e3e19bfd829a01529dbfdeefa8caeb59b3f1b81dadb/large-v3.pt",
    "large": "https://openaipublic.azureedge.net/main/whisper/models/e5b1a55b89c1367dacf97e3e19bfd829a01529dbfdeefa8caeb59b3f1b81dadb/large-v3.pt",
    "large-v3-turbo": "https://openaipublic.azureedge.net/main/whisper/models/aff26ae408abcba5fbf8813c21e62b0941638c5f6eebfb145be0c9839262a19a/large-v3-turbo.pt",
    "turbo": "https://openaipublic.azureedge.net/main/whisper/models/aff26ae408abcba5fbf8813c21e62b0941638c5f6eebfb145be0c9839262a19a/large-v3-turbo.pt",
}
```

在 Windows 系统下模型会被下载到 *%USERPROFILE%\\.cache\whisper* 目录中，比如 `name="small"` 时模型会下载为 *%USERPROFILE%\.cache\whisper\small.pt* 文件。因此我们可以直接复制对应模型的下载路径提前下载好模型。

## 遇到的问题

### 缺少 GPU 警告

```
D:\projects\python\example-video\.venv\Lib\site-packages\whisper\transcribe.py:132: UserWarning: FP16 is not supported on CPU; using FP32 instead
  warnings.warn("FP16 is not supported on CPU; using FP32 instead")
```

可以不用管它。

### 繁体文本问题

模型有的时候会返回繁体文本，需要转换一下。基于 [zhconv](https://pypi.org/project/zhconv/) 实现了 *zhcn_convert.py* 脚本

```python
import zhconv
import argparse
import sys
from pathlib import Path


def convert_to_simplified(input_path, output_path):
    with open(input_path, 'r', encoding='utf-8') as f:
        content = f.read()
    simplified = zhconv.convert(content, 'zh-cn')
    with open(output_path, 'w', encoding='utf-8') as f:
        f.write(simplified)
    print(f"✅ 转换完成：{input_path} -> {output_path}")


def main():
    parser = argparse.ArgumentParser(description="繁简转换工具")
    parser.add_argument("-i", required=True, help="输入文件路径")
    parser.add_argument("-o", required=True, help="输出文件路径")

    args = parser.parse_args()

    if not Path(args.i).exists():
        print(f"❌ 错误：文件不存在：{args.i}")
        sys.exit(1)

    convert_to_simplified(args.i, args.o)


if __name__ == "__main__":
    main()
```
