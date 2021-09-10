---
title: ffmpeg 入门与简单使用
date: 2021-09-10 11:03:23
tags:
- ffmpeg
---

> FFmpeg 是视频处理最常用的开源软件。
它功能强大，用途广泛，大量用于视频网站和商业软件（比如 Youtube 和 iTunes），也是许多音频和视频格式的标准编码/解码实现。

<!-- more -->

# 1. 安装

## centos7 上安装

```bash
sudo yum install epel-release

# 导入存储库GPG密钥并通过安装rpm软件包来启用Nux存储库
sudo rpm -v --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro
sudo rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm

sudo yum install ffmpeg ffmpeg-devel

ffmpeg -version
```
## ubuntu 上安装

```bash
sudo apt -y install ffmpeg

```
# 2. 简介

## 容器
> 视频文件本身其实是一个容器（container），里面包括了视频和音频，也可能有字幕等其他内容.
使用 `ffmpeg -formats` 查看ffmpeg 支持的容器。常见的容器有: mp4, mkv, avi, webm 等。

## 编码格式
视频和音频都需要经过编码，才能保存成文件。不同的编码格式（CODEC），有不同的压缩率，会导致文件大小和清晰度的差异。

查看 FFmpeg 支持的编码格式: `ffmpeg -codecs`

常用的视频编码格式如下:
- 视频
  - 有版权(可以免费使用)
    - H.262
    - H.264
    - H.265
  - 无版权
    - VP8
	- VP9
	- AV1
- 音频
  - mp3
  - aac

上面所有这些都是有损的编码格式，编码后会损失一些细节，以换取压缩后较小的文件体积。无损的编码格式压缩出来的文件体积较大，这里就不介绍了。

## 编码器
> 编码器（encoders）是实现某种编码格式的库文件。只有安装了某种格式的编码器，才能实现该格式视频/音频的编码和解码。

查看 FFmpeg 已安装的编码器: `ffmpeg -encoders`

FFmpeg 内置的编码器:
- 视频
  - libx264：最流行的开源 H.264 编码器
  - NVENC：基于 NVIDIA GPU 的 H.264 编码器
  - libx265：开源的 HEVC 编码器
  - libvpx：谷歌的 VP8 和 VP9 编码器
  - libaom：AV1 编码器
- 音频
  - libfdk-aac
  - aac

# 3. 使用

`ffmpeg [options] [[infile options] -i infile]... {[outfile options] outfile}...`

```bash

ffmpeg \
-y \ # 全局参数
-c:a libfdk_aac -c:v libx264 \ # 输入文件参数
-i input.mp4 \ # 输入文件
-c:v libvpx-vp9 -c:a libvorbis \ # 输出文件参数
output.webm # 输出文件

# 上面的命令将 mp4 文件转成 webm 文件，这两个都是容器格式。输入的 mp4 文件的音频编码格式是 aac，视频编码格式是 H.264；输出的 webm 文件的视频编码格式是 VP9，音频格式是 Vorbis。

```

FFmpeg 常用的命令行参数如下:
```
-c：指定编码器
-c copy：直接复制，不经过重新编码（这样比较快）
-c:v：指定视频编码器
-c:a：指定音频编码器
-i：指定输入文件
-an：去除音频流
-vn： 去除视频流
-preset：指定输出的视频质量，会影响文件的生成速度，有以下几个可用的值 ultrafast, superfast, veryfast, faster, fast, medium, slow, slower, veryslow。
-y：不经过确认，输出时直接覆盖同名文件。

```

## 1. 查看文件信息

```bash
ffprobe -i 01.webm -hide_banner
```

## 2. 转换编码格式

> 转换编码格式（transcoding）指的是， 将视频文件从一种编码转成另一种编码。比如转成 H.264 编码，一般使用编码器libx264，所以只需指定输出文件的视频编码器即可。

```bash
ffmpeg -i 01.webm -c:v libx264 01.mp4
```

## 3. 转换容器格式
> 转换容器格式（transmuxing）指的是，将视频文件从一种容器转到另一种容器。

```bash
# 转码
ffmpeg -i 01.mp4 02.webm

# 流复制
# 其中，-c 是 codec 的简称，表示所有流的编解码器。该命令表示所有流均不进行额外操作，直接复制到新容器中。
ffmpeg -i 01.mp4 -c copy 01.avi
```

## 4. 调整码率
> 调整码率（transrating）指的是，改变编码的比特率，一般用来将视频文件的体积变小。下面的例子指定码率最小为964K，最大为3856K，缓冲区大小为 2000K。

```bash
ffmpeg \
-i input.mp4 \
-minrate 964K -maxrate 3856K -bufsize 2000K \
output.mp4
```

## 5. 改变分辨率（transsizing）
> 只能往低分辨率转

```bash
ffmpeg \
-i input.mp4 \
-vf scale=480:-1 \
output.mp4
```

## 6. 提取流(音频，字幕)

此处的 `-c:a` 表示音频流；视频流 `-c:v` 与字幕流 `-c:s` 自然也类似。 注意：如果音频流与容器冲突时，你需要将 `copy` 改为正确的编解码器（或者删去 `-c:a copy` 来让 FFmpeg 自动选择），以执行重编码。

```bash
# 提取音频
ffmpeg -i video.mp4 -c:a copy audio.aac

# 提取字幕
ffmpeg -i video.mkv -c:s copy subtitle.srt
```

## 7. 添加音轨
> 添加音轨（muxing）指的是，将外部音频加入视频，比如添加背景音乐或旁白。

```bash
# 将音频 01.mp3, 与视频01.mp4 合并成一个文件
ffmpeg -i 01.mp3 -i 01.avi 01-2.avi

```

## 8. 截图

```bash
# 下面的例子是从指定时间开始，连续对1秒钟的视频进行截图。
ffmpeg \
-y \
-i input.mp4 \
-ss 00:01:24 -t 00:00:01 \
output_%3d.jpg

# 如果只需要截一张图，可以指定只截取一帧。
ffmpeg \
-ss 01:23:45 \
-i input \
-vframes 1 -q:v 2 \
output.jpg
# -vframes 1指定只截取一帧，-q:v 2表示输出的图片质量，一般是1到5之间（1 为质量最高）。
```

## 9. 裁剪
裁剪（cutting）指的是，截取原始视频里面的一个片段，输出为一个新视频。可以指定开始时间（start）和持续时间（duration），也可以指定结束时间（end）。

```bash
# ffmpeg -ss [start] -i [input] -t [duration] -c copy [output]
# ffmpeg -ss [start] -to [end] -i [input]  -c copy [output]

# -ss 参数指定了起始的时间戳记
# -t 	参数指定了片段长度（秒）
# -to 指定终止时刻. ( 一定要配置到
# 
ffmpeg -ss 00:00:03 -to 00:00:15 -i 01.avi cut_2.avi

```

## 10. 为音频添加封面
> 有些视频网站只允许上传视频文件。如果要上传音频文件，必须为音频添加封面，将其转为视频，然后上传。

```bash
ffmpeg \
-loop 1 \
-i cover.jpg -i input.mp3 \
-c:v libx264 -c:a aac -b:a 192k -shortest \
output.mp4
# 有两个输入文件，一个是封面图片cover.jpg，另一个是音频文件input.mp3。-loop 1参数表示图片无限循环，-shortest参数表示音频文件结束，输出视频就结束。
```

## 11. 将视频转为gif

```bash
ffmpeg \
  -ss 00:00:10 -to 00:00:20 \
  -i 01.avi \
  -s 640x320 \
  -r 15 out.gif
```

## 12. 左上角加水印

```bash
ffmpeg \
  -i 01.avi -i 01.svg \
  -filter_complex \
  "overlay=40:40" \
  out_6.mp4
# -i 指定输入文件
# overlay 指定水印位置
```

## 13. 将多个图片做成 gif 动图

```bash
cat .img/*.png | ffmpeg -framerate 1 -i - out_01.gif

```

## 14. 合并视频

对于 MPEG 格式的视频，可以直接连接：
```bash
ffmpeg -i "concat:input1.mpg|input2.mpg|input3.mpg" -c copy output.mpg
```

对于非 MPEG 格式容器，但是是 MPEG 编码器（H.264、DivX、XviD、MPEG4、MPEG2、AAC、MP2、MP3 等），可以包装进 TS 格式的容器再合并。在新浪视频，有很多视频使用 H.264 编码器，可以采用这个方法.
```bash
ffmpeg -i input1.flv -c copy -bsf:v h264_mp4toannexb -f mpegts input1.ts
ffmpeg -i input2.flv -c copy -bsf:v h264_mp4toannexb -f mpegts input2.ts
ffmpeg -i input3.flv -c copy -bsf:v h264_mp4toannexb -f mpegts input3.ts

ffmpeg -i "concat:input1.ts|input2.ts|input3.ts" -c copy -bsf:a aac_adtstoasc -movflags +faststart output.mp4

# 将 ts 文件过多时，可使用文件
cat > ./mylist.txt << EOF
file 'input1.ts"
file 'input2.ts"
file 'input3.ts"
EOF

ffmpeg -f concat -i mylist.txt -c copy output.mp4
```



# 参考

- [FFmpeg 视频处理入门教程](https://www.ruanyifeng.com/blog/2020/01/ffmpeg.html)

