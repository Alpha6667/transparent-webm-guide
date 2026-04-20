# 透明视频压缩为 WebM 的方法、验证方式与 Demo

## 目标

本文整理以下内容：

- 如何把带透明通道的视频压缩为透明 `WebM`
- 如何验证压缩结果是否仍然是 `WebM`
- 如何验证透明底是否真的保留成功
- 一个最小可运行的网页 Demo

## 前提条件

- 源文件本身必须真的带 alpha 通道
- 常见输入格式：透明 `mov`，例如 `ProRes 4444`
- 需要本机安装 `ffmpeg` 和 `ffprobe`

## 推荐压缩命令

推荐使用 `VP9 + WebM + alpha`：

```bash
ffmpeg -i input.mov \
  -c:v libvpx-vp9 \
  -pix_fmt yuva420p \
  -auto-alt-ref 0 \
  -b:v 0 \
  -crf 32 \
  output.webm
```

参数说明：

- `-c:v libvpx-vp9`
  - 使用 `VP9` 编码
- `-pix_fmt yuva420p`
  - 保留 alpha 通道
- `-auto-alt-ref 0`
  - 透明视频常用设置，避免部分 alpha 异常
- `-b:v 0`
  - 使用 CRF 模式控制质量
- `-crf 32`
  - 压缩强度较高，适合明显缩小体积

质量调节建议：

- 更清晰：`-crf 24` 到 `-crf 28`
- 平衡体积与质量：`-crf 30` 到 `-crf 32`
- 更小体积：`-crf 34` 到 `-crf 36`

## 验证是否成功压成 WebM

执行：

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream=codec_name,pix_fmt:format=format_name \
  -of default=noprint_wrappers=1 \
  output.webm
```

期望看到类似结果：

```text
codec_name=vp9
pix_fmt=yuva420p
format_name=matroska,webm
```

结果解释：

- `codec_name=vp9`
  - 视频编码是 `VP9`
- `pix_fmt=yuva420p`
  - 像素格式带 alpha
- `format_name=matroska,webm`
  - 容器为 `WebM`

## 验证是否带 alpha 标记

执行：

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream_tags=alpha_mode \
  -of default=noprint_wrappers=1 \
  output.webm
```

可能看到：

```text
TAG:alpha_mode=1
```

这说明文件已经写入 alpha 标记。

注意：

- 只有 `alpha_mode=1` 还不够
- 最好继续验证 alpha 平面里是否真的有透明像素

## 验证透明底是否真的保留

执行：

```bash
ffmpeg -i output.webm \
  -vf alphaextract,signalstats \
  -an -f null -
```

关注输出中的统计值，例如：

```text
YMIN=0
YMAX=255
```

判断方式：

- `YMIN=0`
  - 说明存在完全透明像素
- `YMAX=255`
  - 说明存在完全不透明像素

如果两者都出现，通常说明 alpha 平面是真实存在的，不是只有标记而没有实际透明内容。

更稳妥的做法：

- 对源文件执行一次同样命令
- 对压缩后的 `webm` 再执行一次
- 对比两边 alpha 统计是否仍然合理

## 最小网页 Demo

把下面内容保存为 `preview.html`，并把 `output.webm` 放在同目录：

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>Transparent WebM Demo</title>
  <style>
    body {
      margin: 0;
      padding: 24px;
      background: #111;
      color: #fff;
      font-family: sans-serif;
    }

    .row {
      display: flex;
      gap: 24px;
      flex-wrap: wrap;
    }

    .panel {
      width: 360px;
    }

    .box {
      width: 360px;
      height: 202px;
      overflow: hidden;
      border-radius: 12px;
      border: 1px solid #333;
    }

    .black {
      background: #000;
    }

    .checker {
      background:
        linear-gradient(45deg, #ccc 25%, transparent 25%) -10px 0/20px 20px,
        linear-gradient(-45deg, #ccc 25%, transparent 25%) -10px 0/20px 20px,
        linear-gradient(45deg, transparent 75%, #ccc 75%) -10px 0/20px 20px,
        linear-gradient(-45deg, transparent 75%, #ccc 75%) -10px 0/20px 20px,
        #fff;
    }

    video {
      width: 100%;
      height: 100%;
      object-fit: contain;
      display: block;
    }
  </style>
</head>
<body>
  <h1>透明 WebM 验证</h1>
  <div class="row">
    <div class="panel">
      <p>黑底</p>
      <div class="box black">
        <video autoplay muted loop playsinline>
          <source src="./output.webm" type="video/webm" />
        </video>
      </div>
    </div>

    <div class="panel">
      <p>棋盘格底</p>
      <div class="box checker">
        <video autoplay muted loop playsinline>
          <source src="./output.webm" type="video/webm" />
        </video>
      </div>
    </div>
  </div>
</body>
</html>
```

验证标准：

- 如果棋盘格背景能从主体外围透出来，说明透明显示正常
- 如果黑底和棋盘格看起来几乎一样，通常说明当前播放器链路没有正确显示 alpha

## Android Demo 方案说明

如果需要在 Android 上做最小验证 Demo，当前更适合走：

- `WebView + file:///android_asset + 本地 transparent WebM`

原因：

- 浏览器链路更容易正确显示 `VP9 + alpha`
- 原生播放器如 `ExoPlayer` 通常能播视频，但透明区域经常表现为黑底

因此：

- 适合验证透明显示：`WebView`
- 不适合稳定透明方案验证：`ExoPlayer`

## 最小工作流

### 1. 压缩

```bash
ffmpeg -i input.mov -c:v libvpx-vp9 -pix_fmt yuva420p -auto-alt-ref 0 -b:v 0 -crf 32 output.webm
```

### 2. 看编码和容器

```bash
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name,pix_fmt:format=format_name -of default=noprint_wrappers=1 output.webm
```

### 3. 看 alpha 标记

```bash
ffprobe -v error -select_streams v:0 -show_entries stream_tags=alpha_mode -of default=noprint_wrappers=1 output.webm
```

### 4. 看透明像素是否真实存在

```bash
ffmpeg -i output.webm -vf alphaextract,signalstats -an -f null -
```

## 工程结论

- 透明 `WebM` 推荐使用 `VP9`
- 压制时重点参数是 `libvpx-vp9`、`yuva420p`、`-auto-alt-ref 0`
- 验证时不要只看文件后缀
- 至少要同时验证：
  - 容器是不是 `WebM`
  - 编码是不是 `VP9`
  - 是否带 alpha 标记
  - alpha 平面里是否真的有透明像素
- 网页和 `WebView` 更适合做透明 `WebM` 验证
- `ExoPlayer` 不适合当透明 `WebM` 的稳定播放方案
