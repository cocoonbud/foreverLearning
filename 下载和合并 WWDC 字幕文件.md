# 下载和合并 Apple WWDC 字幕文件

[TOC]

## 前言

作为一名资深的 iOS 开发人员，每年 WWDC 后的官方视频内容学习总结对于保持持续学习还是比较有帮助的。为此写了一段 Python 脚本来比较高效的下载和合并 WWDC 视频的字幕文件。

## 分析

以 [分析堆内存](https://developer.apple.com/cn/videos/play/wwdc2024/10173/) 为例：
<img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//image-20240624103434488.png" alt="image-20240624103434488" style="zoom:50%;" />
图中的`.webvtt` 格式的文件就是我们需要的字幕文件。
<img src="https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//image-20240624104139506.png" alt="image-20240624104139506" style="zoom:50%;" />
这个 `prog_index_m3u8` 文件里，我们可以看到此视频有多少个 `.webvtt` 文件个数。接下来就是解决如何下载每一个 `.webvtt` 文件并合并成一个完整的字幕文本。

## 下载脚本
### 1. 脚本功能概述
这段 Python 脚本实现的功能：
- 并行下载多个 WWDC 视频的字幕文件。
- 合并下载的字幕文件并清理冗余信息，合并成一个完整的字幕文件。
### 2. 脚本内容
**输入 URL 和文件数量**
首先，脚本会提示用户输入基础 URL 和字幕文件的总数：

```python
base_url = input("Enter the base URL (e.g., https://devstreaming-cdn.apple.com/videos/wwdc/2024/10173/4/xx-xx-xx-xx-xx/subtitles/eng/sequence_0.webvtt): ").strip()
count = int(input("Enter the total number of subtitle files: ").strip())

if not base_url.endswith(".webvtt"):
    raise ValueError("The entered URL is incorrect. Ensure it ends with .webvtt.")
```
**下载单个字幕文件**
下载字幕文件的函数 download_subtitle 会根据文件数量生成相应的 URL，并尝试下载文件：

```python
# Download single subtitle file
def download_subtitle(counter):
    url = f"{base_url_prefix}_{counter}.webvtt"
    response = requests.get(url)
    if response.status_code == 200:
        file_path = f"sequence_{counter}.webvtt"
        with open(file_path, "wb") as file:
            file.write(response.content)
        print(f"Downloaded {file_path}")
        return file_path
    else:
        print(f"Failed to download sequence_{counter}.webvtt")
        return None
```
**并行下载字幕文件**
为了提高下载效率，脚本使用 ThreadPoolExecutor 并行下载字幕文件：
```python
# Concurrent download
def download_subtitles_concurrently(count):
    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = [executor.submit(download_subtitle, counter) for counter in range(count)]
        return [future.result() for future in as_completed(futures)]
```
**合并和清理字幕文件**
下载完成后，脚本会将所有字幕文件合并为一个完整的文件。但是 webvtt 格式的字幕文件里有时间轴、格式信息还有一些空行，所以在此也一并使用正则表达式清理冗余信息。
![image-20240624161513499](https://cdn.jsdelivr.net/gh/cocoonbud/TyporaPic@master//image-20240624161513499.png)

```python
# Merge and clean subtitles
def merge_and_clean_subtitles(file_paths):
    with open("full.webvtt", "wb") as full_file:
        for file_path in file_paths:
            if file_path and os.path.exists(file_path):
                with open(file_path, "rb") as file:
                    content = file.read()
                    # Use regular expressions to remove "WEBVTT" tags and timestamp lines
                    cleaned_content = re.sub(rb'(WEBVTT\n+)|(\d{2}:\d{2}:\d{2}\.\d{3} --> \d{2}:\d{2}:\d{2}\.\d{3}\n)|(\n{2,})', b'', content)
                    full_file.write(cleaned_content)
                os.remove(file_path)
                print(f"Merged {file_path}")
```
### 3. github 完整代码
[github 完整代码](https://github.com/cocoonbud/wwdc-tools-notes/tree/master/tools)

## 尾声

通过上述脚本，可以高效的下载和合并 WWDC 视频的字幕文件，为后续学习提供便利。希望能给大家带来便利。