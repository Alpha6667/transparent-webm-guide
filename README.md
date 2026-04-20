# Transparent WebM Encoding and Validation Guide

[中文说明](./README.zh-CN.md)

本仓库用于沉淀透明视频压缩为 `WebM` 的编码方法、alpha 保留验证流程，以及最小化验证 Demo 的技术说明。

## Repository Contents

- `transparent-webm-guide.md`
  - 完整技术说明
  - 包含编码命令、验证命令、透明底验证方法和网页 Demo
- `transparent-webm-quickstart.md`
  - 快速操作手册
  - 适合在协作场景中直接复用

## Scope

- 源视频本身带 alpha 通道
- 需要输出透明 `WebM`
- 需要验证输出文件是否仍然保留透明底

## Recommended Encoding Profile

推荐组合：

- 容器：`WebM`
- 编码：`VP9`
- 像素格式：`yuva420p`

参考命令：

```bash
ffmpeg -i input.mov \
  -c:v libvpx-vp9 \
  -pix_fmt yuva420p \
  -auto-alt-ref 0 \
  -b:v 0 \
  -crf 32 \
  output.webm
```

## Validation Checklist

### 1. Validate Container and Codec

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream=codec_name,pix_fmt:format=format_name \
  -of default=noprint_wrappers=1 \
  output.webm
```

### 2. Validate Alpha Metadata

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream_tags=alpha_mode \
  -of default=noprint_wrappers=1 \
  output.webm
```

### 3. Validate Real Alpha Pixels

```bash
ffmpeg -i output.webm \
  -vf alphaextract,signalstats \
  -an -f null -
```

## Notes

- 浏览器和 `WebView` 更适合作为透明 `WebM` 的验证环境
- `ExoPlayer` 对透明 `WebM` 的显示通常不稳定，常见现象是黑底
