# Transparent WebM Notes

这个仓库用于整理透明视频压缩为 `WebM` 的方法、验证步骤和快速操作说明。

## 文件说明

- `transparent-webm-guide.md`
  - 完整版说明
  - 包含压缩命令、验证命令、透明底验证方法和网页 Demo
- `transparent-webm-quickstart.md`
  - 精简版速查
  - 适合同事快速照着执行

## 适用场景

- 源视频本身带 alpha 通道
- 需要压缩为透明 `WebM`
- 需要验证输出文件是否仍然保留透明底

## 推荐方案

推荐使用：

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

## 快速验证

### 验证容器和编码

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream=codec_name,pix_fmt:format=format_name \
  -of default=noprint_wrappers=1 \
  output.webm
```

### 验证 alpha 标记

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream_tags=alpha_mode \
  -of default=noprint_wrappers=1 \
  output.webm
```

### 验证透明像素是否真实存在

```bash
ffmpeg -i output.webm \
  -vf alphaextract,signalstats \
  -an -f null -
```

## 说明

- 浏览器和 `WebView` 更适合验证透明 `WebM`
- `ExoPlayer` 对透明 `WebM` 的显示通常不稳定，常见现象是黑底
