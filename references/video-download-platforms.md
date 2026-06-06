# Video Download: Platform Comparison (2026-05-30 实测)

## 三平台下载方案对比

| 平台 | 推荐命令 | 成功率 | 关键条件 |
|---|---|---|---|
| **Bilibili** | `yt-dlp --cookies-from-browser chrome --no-playlist` | ⭐⭐⭐⭐⭐ | Chrome 已登录 B站 |
| **YouTube** | `HTTP_PROXY=... yt-dlp --cookies-from-browser chrome` | ⭐⭐⭐⭐ | HTTP 代理(7890) + Chrome 已登录 YouTube |
| **抖音** | Camoufox `browser_console: document.querySelector('video')?.src` → curl | ⭐⭐⭐ | 无需登录，但 DRM 视频失效 |

---

## Bilibili 完整下载示例（2026-05-30 实测）

视频：BV17xo9BsEnx（Hermes Agent 介绍，14.7万播放，13分11秒）

```bash
# 创建工作目录
mkdir -p /tmp/bilibili_video
cd /tmp/bilibili_video

# 一条命令搞定视频+字幕+弹幕
yt-dlp --cookies-from-browser chrome \
  --write-subs --write-auto-subs --sub-lang ai-zh \
  --write-comments \
  --no-playlist \
  -o "video.%(ext)s" \
  "https://b23.tv/qMpnJbF"

# 输出：
# video.mp4          (76MB, 1080P+音频合并)
# video.ai-zh.srt    (1756行字幕)
# video.comments.json (267条弹幕)
# video.info.json    (元数据)
```

**注意**：`--cookies-from-browser chrome` 自动从 Chrome 实时提取 1213 个 cookies，无需手动导出 cookies 文件。

---

## YouTube 完整下载示例

```bash
# 必须先设置 HTTP 代理（socks5 7891 口不行，要用 HTTP 7890 口）
export HTTP_PROXY="http://127.0.0.1:7890"

yt-dlp --cookies-from-browser chrome \
  --write-subs --write-auto-subs --sub-lang zh-Hans,en \
  --no-playlist \
  -o "/tmp/yt_output.%(ext)s" \
  "https://youtu.be/szPfDAwLcT8"

# 下载内容：
# - 视频文件（153MB）
# - 中文字幕（vtt/srt）
# - 英文字幕
```

**国内 IP 问题**：不挂代理直接跑 yt-dlp 会报 `Sign in to confirm you're not a bot`。

---

## 抖音 Camoufox 提取方案

### 流程
```
browser_navigate("https://v.douyin.com/xxxxx")
↓
browser_console("document.querySelector('video')?.src")
↓
提取 v11-weba.douyinvod.com 直链
↓
curl -L -o output.mp4
↓
FFmpeg 合并视频+音频（抖音分开存储）
```

### 关键发现
- 抖音视频直链是真实 CDN URL（`v11-weba.douyinvod.com`），不是 blob://，可直接下载
- 视频+音频分开存储，下载后必须 FFmpeg 合并再使用
- 若 Camoufox 打开页面显示"你要观看的视频不存在" = 视频已下架/私有化，所有方案失效，切换方案A（用户直接发文件）

### 已验证失效的第三方解析（2026-05 实测）
- tikwm.com — Cloudflare 403
- tiktokdownload.online — 301 → Cloudflare
- douyinparse.com — 无响应

---

## 截图技巧

```bash
# 从视频中每秒抽1帧（用于快速浏览选图）
ffmpeg -i video.mp4 -vf "fps=1" -q:v 2 frames/frame_%03d.jpg

# 精确截取单帧（用于关键画面）
ffmpeg -ss 00:05:18 -i video.mp4 -vframes 1 -q:v 2 output.jpg

# 视频信息
ffprobe -v error -show_entries format=duration -of csv=p=0 video.mp4
```

---

## 踩坑记录

### 坑1：B站下载到整个系列（14集全下）
**原因**：未加 `--no-playlist`，BV 号是合集
**解决**：加上 `--no-playlist` 只下当前视频

### 坑2：YouTube 下载报 "Sign in to confirm you're not a bot"
**原因**：国内 IP 被 YouTube 封锁
**解决**：设置 `HTTP_PROXY="http://127.0.0.1:7890"`（socks5 不兼容）

### 坑3：Whisper 把"银子"识别成"星号"
**原因**：Whisper 对中文专有名词识别差，"銀"字形近误识
**解决**：转写后人工校验，或用 B站 AI 字幕（SRT格式）更准确

### 坑4：yt-dlp Chrome cookies 对抖音失效
**原因**：抖音 DRM 要求实时会话 cookie，Chrome 保存的 cookie 无法满足
**解决**：方案A（用户直接分享文件）或方案B（跳过原版复刻风格）