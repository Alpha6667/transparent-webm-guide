# 透明 WebM 压缩与验证速查

## 适用场景

- 源视频本身带透明通道
- 目标是压成透明 `WebM`
- 需要快速确认是否真的保留透明底

## 1. 压缩命令

```bash
ffmpeg -i input.mov \
  -c:v libvpx-vp9 \
  -pix_fmt yuva420p \
  -auto-alt-ref 0 \
  -b:v 0 \
  -crf 32 \
  output.webm
```

常用说明：

- 编码：`VP9`
- 容器：`WebM`
- 透明：`yuva420p`
- 推荐质量范围：`crf 28` 到 `crf 32`

## 2. 验证是不是压成功了

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream=codec_name,pix_fmt:format=format_name \
  -of default=noprint_wrappers=1 \
  output.webm
```

期望结果：

```text
codec_name=vp9
pix_fmt=yuva420p
format_name=matroska,webm
```

## 3. 验证有没有 alpha 标记

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream_tags=alpha_mode \
  -of default=noprint_wrappers=1 \
  output.webm
```

期望结果：

```text
TAG:alpha_mode=1
```

## 4. 验证透明底是不是真的还在

```bash
ffmpeg -i output.webm \
  -vf alphaextract,signalstats \
  -an -f null -
```

关注输出里是否出现：

```text
YMIN=0
YMAX=255
```

判断：

- 有 `YMIN=0`：说明存在透明像素
- 有 `YMAX=255`：说明存在不透明像素
- 两个都有：通常说明透明底保留成功

## 5. 网页快速预览

最简单的判断方法：

- 把视频放在黑底上看一遍
- 再放在棋盘格背景上看一遍
- 如果棋盘格能从主体外围透出来，说明透明正常

## 6. 使用建议

- 验证透明视频，优先用浏览器或 `WebView`
- 不要只看文件后缀，要看 `ffprobe` 结果
- 不要只看 `alpha_mode=1`，要再看 alpha 平面统计
- `ExoPlayer` 播放透明 `WebM` 常见现象是黑底，不适合当稳定透明方案

## 7. 最小结论

- 压缩命令没问题：看 `codec_name=vp9`
- 文件确实是 `WebM`：看 `format_name=matroska,webm`
- 文件带 alpha：看 `pix_fmt=yuva420p` 和 `TAG:alpha_mode=1`
- 透明真实存在：看 `YMIN=0` 和 `YMAX=255`
