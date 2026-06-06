---
name: video-knowledge-enricher
description: "Video notes enrichment: ingest video URL → extract transcript → deep research on concepts → capture key video screenshots → cross-reference existing vault → produce comprehensive Obsidian notes with full video content fidelity."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [video, notes, knowledge-base, enrichment, obsidian, bilibili, youtube, screenshots]
    category: productivity
    related_skills: [video-content-extraction, obsidian]
    requires_toolsets: [terminal, browser, web]
---

# Video Knowledge Enricher

Solve the "shallow notes" problem: video notes that only capture what was said,
but miss background knowledge, related concepts, connections to existing notes,
and visual references from the video itself.

**Core insight (from nashsu/llm_wiki's two-step ingestion):**
- Step 1 (Analyze): LLM reads source material → identifies what is said, what's implied, what's missing
- Step 2 (Enrich): LLM researches the gaps → generates comprehensive notes with context + screenshots

This skill applies that pattern specifically to video learning, with screenshot capture.

## The Problem with Typical Video Notes

Typical flow:
```
Watch video → Write notes from memory → Store in Obsidian
```

**What goes wrong:**
- Your note depth = your current understanding at viewing time
- Concepts you're unfamiliar with get one-line mentions you can't expand on later
- No connections to related things you already know
- No background context that would deepen understanding
- Good ideas mentioned in passing get lost
- Video's visual content (diagrams, charts, examples) is completely absent

## The Enrichment Pipeline

```
Video URL / File
    ↓
[STEP 1] Download Video (720p for screenshots)
    ↓
[STEP 2] Extract — video-content-extraction skill
    ↓  (transcript/subtitles + video file)
Raw transcript + video
    ↓
[STEP 3] Analyze — identify knowledge gaps + key visual moments
    ↓
List of: concepts needing expansion + screenshot timestamps
    ↓
[STEP 4] Capture Screenshots — extract key frames
    ↓
Timestamped screenshots saved to vault assets/
    ↓
[STEP 5] Deep Research — web search each concept
    ↓
Synthesized background knowledge per concept
    ↓
[STEP 6] Cross-Reference — search existing Obsidian vault
    ↓
Links to related existing notes
    ↓
[STEP 7] Synthesize — generate comprehensive notes
    ↓
Rich Obsidian note with:
  - Video's actual content (faithful)
  - Key screenshots at timestamps (visual fidelity)
  - Background context (from research)
  - Related concepts (from cross-reference)
  - Connections to existing vault (wikilinks)
  - Provenance (sources cited)
```

## When This Skill Activates

Use this skill when the user:
- Shares a video URL (YouTube, Bilibili) and wants comprehensive notes
- Has existing shallow video notes that need enrichment
- Wants notes that go beyond "what the video said" into "what it means and connects to"
- Wants visual references from the video preserved in the notes

## Step-by-Step Instructions

### Step 1 — Download Video (for Screenshots)

**YouTube (needs proxy + Chrome cookies):**
```bash
# Required: HTTP proxy (Clash at port 7890) + Chrome logged into YouTube
HTTP_PROXY="http://127.0.0.1:7890" \
  yt-dlp --cookies-from-browser chrome \
    -f "best[height<=720]" \
    -o "/tmp/enrich_video.%(ext)s" \
    "https://youtube.com/watch?v=VIDEO_ID"
```

**Bilibili (Chrome cookies — --no-playlist to avoid downloading entire series):**
```bash
yt-dlp --cookies-from-browser chrome \
  -f "best[height<=720]" \
  --no-playlist \
  -o "/tmp/enrich_video.%(ext)s" \
  "https://www.bilibili.com/video/BV_ID"
```

**抖音（Camoufox 提取直链）：**
```bash
# Step 1: browser_navigate 打开链接
# Step 2: browser_console 执行: document.querySelector('video')?.src
# Step 3: curl 下载视频直链（抖音视频+音频分开存储，需用 FFmpeg 合并）
```

### Step 2 — Extract Video Content

Use the `video-content-extraction` skill to get the transcript.

**YouTube (with proxy + Chrome cookies):**
```bash
HTTP_PROXY="http://127.0.0.1:7890" \
  yt-dlp --cookies-from-browser chrome \
    --write-subs --write-auto-subs --sub-lang zh-Hans,en \
    --no-playlist -o "/tmp/enrich_subs.%(ext)s" \
    "https://youtube.com/watch?v=VIDEO_ID"
```

**Bilibili (Chrome cookies — gets AI subtitles + danmaku automatically):**
```bash
yt-dlp --cookies-from-browser chrome \
  --write-subs --write-auto-subs --sub-lang ai-zh \
  --write-comments \
  --no-playlist \
  -o "/tmp/enrich_video.%(ext)s" \
  "https://www.bilibili.com/video/BV_ID"
```
This downloads: video file + AI-generated Chinese SRT subtitle + up to 267 danmaku comments.

If no subtitles available (rare on Bilibili with login), transcribe with faster-whisper:
```bash
~/opt/anaconda3/bin/python -c "
from faster_whisper import WhisperModel
model = WhisperModel('base', device='cpu', compute_type='int8')
segments, info = model.transcribe('/tmp/enrich_video.mp4', language='zh', beam_size=5)
with open('/tmp/transcript.txt', 'w') as f:
    for seg in segments:
        f.write(f'[{seg.start:.1f}s->{seg.end:.1f}s] {seg.text}\n')
print(f'Transcribed {len(list(segments))} segments, language={info.language}')
"
```

> **Note**: faster-whisper runs under conda env at `~/opt/anaconda3/bin/python`. Whisper may misrecognize some Chinese terms (e.g., "银子" → "星号", "门庭若市" → "门停若是"). Preview transcript and correct obvious errors before generating notes.

### Step 3 — Analyze for Knowledge Gaps + Screenshot Moments

Read the full transcript, then produce an **Analysis Report**:

**A. Key Concepts (what the video teaches)**
List each distinct concept/term/idea. Rate each:
- `familiar` — you already understand this well
- `partial` — you know a little but details are fuzzy
- `new` — completely new concept you can't explain yet

**B. Knowledge Gaps**
For each `partial` or `new` concept:
- What the video says about it (brief)
- What the video DOESN'T say but a learner needs to know
- What questions would a curious learner ask?

**C. Key Screenshot Moments**
From the transcript, identify moments that need visual capture:
```
Timestamp | Why This Frame Matters
00:02:34  | 核心概念图示（护城河模型图）
00:05:18  | 案例：茅台财务数据截图
00:08:45  | 对比表格（两种商业模式对比）
00:12:30  | 关键结论的文字版（方便引用）
00:15:00  | 总结要点思维导图风格
```
Aim for 5-10 key screenshots per 15-20 min video.

**D. Cross-Reference Candidates**
Search Obsidian vault for each key concept:
```bash
OBSIDIAN_VAULT="${OBSIDIAN_VAULT:-$HOME/Documents/Obsidian Vault}"
grep -rl "keyword" "$OBSIDIAN_VAULT" --include="*.md" -l
```

**E. Research Agenda**
Convert knowledge gaps into web search queries.

### Step 4 — Capture Screenshots

```bash
# Create screenshot directory
SCREENSHOT_DIR="$OBSIDIAN_VAULT/assets/video-screenshots/$(date +%Y%m%d_%H%M%S)/"
mkdir -p "$SCREENSHOT_DIR"

# Method A: Specific timestamps (most precise)
ffmpeg -ss 00:02:34 -i /tmp/enrich_video.mp4 \
  -vframes 1 -q:v 2 "$SCREENSHOT_DIR/00_02_34_concept_diagram.jpg"

ffmpeg -ss 00:05:18 -i /tmp/enrich_video.mp4 \
  -vframes 1 -q:v 2 "$SCREENSHOT_DIR/00_05_18_maotai_data.jpg"

# Method B: Automated interval (for review and selection)
# Pick every N seconds based on video length
# For 10min video: every 30 seconds = 20 screenshots for review
INTERVAL=30  # seconds
ffmpeg -i /tmp/enrich_video.mp4 \
  -vf "fps=1/$INTERVAL,scale=1920:1080" \
  "$SCREENSHOT_DIR/frame_%04d.jpg"

# Method C: Scene change detection (only capture meaningful frames)
ffmpeg -i /tmp/enrich_video.mp4 \
  -vf "select='gt(scene,0.3)',scale=1280:720" \
  -vsync vfr \
  "$SCREENSHOT_DIR/scene_%04d.jpg"

# Rename key screenshots to descriptive names
mv /tmp/enrich_video/frame_0010.jpg "$SCREENSHOT_DIR/00_05_18_maotai_financial_table.jpg"
mv /tmp/enrich_video/frame_0015.jpg "$SCREENSHOT_DIR/00_08_45_business_model_comparison.jpg"
```

**Screenshot quality tips:**
- Use `-q:v 2` for high-quality JPEG (lower number = higher quality)
- `scale=1920:1080` for full HD; `scale=1280:720` for smaller file size
- Prefer specific timestamps from transcript analysis over random sampling
- Key moments: concept diagrams, data tables, comparison charts, conclusion text

### Step 5 — Deep Research (Parallel Web Searches)

For each concept rated `partial` or `new`, run web searches:
```bash
# Parallel searches
search "企业护城河分析框架" &
search "巴菲特护城河概念原文" &
search "消费股护城河案例 茅台" &
wait
```

Synthesize findings into 2-3 sentence summaries per concept.

### Step 6 — Obsidian Cross-Reference

```bash
OBSIDIAN_VAULT="${OBSIDIAN_VAULT:-$HOME/Documents/Obsidian Vault}"
grep -rl "护城河\|经济壁垒" "$OBSIDIAN_VAULT" 2>/dev/null
grep -rl "贵州茅台\|估值" "$OBSIDIAN_VAULT" 2>/dev/null
```

Build wikilink list for the new note.

### Step 7 — Synthesize Comprehensive Notes

Generate the final Obsidian note:

```markdown
---
title: [视频标题]
source: [YouTube/Bilibili URL]
type: video-notes
created: {{date}}
updated: {{date}}
tags: [video, [topic], [subtopic]]
video_metadata:
  duration: "[e.g., 14分07秒]"
  author: [author name]
  platform: [YouTube/Bilibili]
  date: [video publish date]
  url: [original URL]
enriched: true
screenshots: true
---

# [视频标题]

> [!INFO]- 一句话总结
> [What you'll be able to do after watching — be specific and actionable]

---

## 🎯 核心知识点

### [Concept 1 — 给它一个描述性标题]

**视频说了什么：**
[What the video explicitly states, in your own words]

**背景扩展：**
[From web research — 2-3 sentences of background context]

**相关截图：**
![概念图示](assets/video-screenshots/TIMESTAMP_concept_diagram.jpg)
*来源：视频 02:34*

**我的理解：**
[Your personal understanding after research]

**相关资源：**
- 关联笔记：[[existing-note-name]]
- 参考来源：[Web source](url)

---

### [Concept 2]

... (same structure)

---

## 🔗 与现有知识的关联

| 新概念 | 现有笔记 | 如何关联 |
|--------|----------|----------|
| [[新页面-概念A]] | [[已有笔记-茅台]] | 茅台的护城河分析正好对应视频中的品牌护城河类型 |
| [[新页面-概念B]] | [[已有笔记-估值]] | 视频中的现金流折现法与现有估值笔记互为补充 |

## ❓ 仍存在的疑问

| 问题 | 状态 | 寻找方向 |
|------|------|----------|
| [未解答的问题] | open | [What to research next] |

---

## 📺 视频完整笔记（时间戳导航）

> [!NOTE]- 视频章节导航
> 点击下方时间戳可直接跳转到视频对应位置

### [00:00-02:00] 开场介绍
![开场截图](assets/video-screenshots/TIMESTAMP_intro.jpg)
**核心内容：** [关键信息]

### [02:00-05:00] 概念讲解
![概念截图](assets/video-screenshots/TIMESTAMP_concept.jpg)
**核心内容：** [关键信息]
> 💡 *视频原话："[值得引用的原文]" — 02:34*

### [05:00-08:00] 案例分析：茅台
![茅台案例截图](assets/video-screenshots/TIMESTAMP_maotai.jpg)
**核心内容：** [关键信息]

### [08:00-10:00] 案例分析：美的
![美的案例截图](assets/video-screenshots/TIMESTAMP_midea.jpg)
**核心内容：** [关键信息]

### [10:00-12:00] 对比与总结
![对比表格截图](assets/video-screenshots/TIMESTAMP_comparison.jpg)
**核心内容：** [关键信息]

### [12:00-14:07] 最终结论
![结论截图](assets/video-screenshots/TIMESTAMP_conclusion.jpg)
**核心内容：** [关键信息]
> ⭐ *核心结论：[一句话总结]*

---

## 🖼️ 视频关键画面收藏

### 概念图示类
![护城河模型图](assets/video-screenshots/TIMESTAMP_moat_model.jpg)
*护城河分析模型 | 视频 02:34*

![竞争格局图](assets/video-screenshots/TIMESTAMP_competition.jpg)
*行业竞争格局分析 | 视频 05:45*

### 数据图表类
![茅台财务数据](assets/video-screenshots/TIMESTAMP_maotai_financial.jpg)
*茅台近5年财务摘要 | 视频 06:12*

![对比数据表](assets/video-screenshots/TIMESTAMP_data_table.jpg)
*茅台vs美的关键指标对比 | 视频 08:30*

### 结论文字类
![核心结论](assets/video-screenshots/TIMESTAMP_key_conclusion.jpg)
*"[关键结论原话]" | 视频 11:20*

![行动建议](assets/video-screenshots/TIMESTAMP_action.jpg)
*"[可操作的建议]" | 视频 12:45*

---

## 📊 概念对比表

> [!TIP]- 视频中的对比内容整理

| 维度 | 茅台 | 美的 | 说明 |
|------|------|------|------|
| 行业 | 高端白酒 | 消费电器 |  |
| 护城河类型 | 品牌+文化 | 成本+渠道 |  |
| 毛利率 | ~92% | ~25% |  |
| 净资产收益率 | >30% | >25% |  |

---

## 📚 参考来源

- 视频字幕/转录（原始材料）：[URL]
- [Web source 1](url) — [why it's relevant]
- [Web source 2](url) — [why it's relevant]
- 视频截图（按时间戳截取，共N张）

---

## 🔍 自我验证：笔记质量检查清单

在生成完笔记后、交付用户前，必须执行以下检查，并修复发现的问题：

### 检查 1：内容完整性（Content Completeness）

对照原始字幕逐段检查：

```
□ 每章节的核心内容都有记录（不是"空手套白狼"）
□ 视频中出现的每一个概念都出现在笔记里
□ 数据/数字/日期等具体信息准确无误（对着字幕核对）
□ 关键引语/原话都有标注时间戳
□ 案例分析（茅台、美的等）都有记录
```

**验证方法：** 播放对应时间戳，确认笔记内容与视频匹配。

### 检查 2：截图质量（Screenshot Quality）

```
□ 截图清晰度：文字可读，无严重模糊
□ 截图相关性：每张截图都是因为有内容价值，不是凑数
□ 截图完整性：关键画面（图表/数据/结论）都有截取
□ 截图标注：每张截图都注明了时间戳和"为什么截这张"
```

**验证方法：** 快速浏览所有截图，确认无"不知道为什么要截"的废图。

### 检查 3：知识准确性（Factual Accuracy）

```
□ 概念定义是否准确（与视频原话对照）
□ 数据引用是否正确（数字、单位、日期）
□ 逻辑是否连贯（上下段落之间因果关系成立）
□ 无幻觉（笔记里写了视频没说的事）
```

**发现错误时的处理：**
```markdown
⚠️ [待核实] 视频 05:23 处提到"毛利率92%"，此处描述可能有误，
   需回放原视频确认是92%还是92%附近的某个具体数字。
   → 已标注，待用户核实后更正。
```

### 检查 4：交叉引用（Cross-Reference）

```
□ 每个核心概念都在 Obsidian 里有 wikilink（或新建页面）
□ 关联到现有笔记的地方都正确
□ "仍存在的疑问"有真实内容，不是空行
□ 没有孤立页面（完全没有 wikilinks 的页面）
```

**验证命令：**
```bash
# 列出笔记中所有 wikilinks
grep -o '\[\[[^]]*\]\]' /path/to/note.md | sort -u

# 验证每个 wikilink 对应的文件是否存在
for link in $(grep -o '\[\[[^]]*\]\]' note.md | tr -d '[]'); do
  if [ ! -f "$OBSIDIAN_VAULT/${link}.md" ] && [ ! -f "$OBSIDIAN_VAULT/${link}/index.md" ]; then
    echo "⚠️ 孤立链接: [[$link]] — 无对应文件"
  fi
done
```

### 检查 5：格式规范性（Format Compliance）

```
□ frontmatter 完整（title, created, updated, tags, source, enriched）
□ 时间戳格式正确（00:00:00 或 HH:MM:SS）
□ 表格结构整齐（无错位）
□ 无乱码/无emoji缺失
□ 截图路径使用相对路径（assets/...）而非绝对路径
```

### 检查 6：可读性（Readability）

```
□ 笔记可以在 10 分钟内通读完全文
□ 每个章节都有清晰的标题
□ 核心结论在笔记里突出显示（> [!INFO] 或 ⭐）
□ 没有一大段文字堆在一起（超过5行就分段）
```

### 修复操作（当发现问题时的处理）

```python
# Python 辅助检查脚本（save as scripts/self_check.py）
import re
import sys

def check_note(filepath):
    errors = []
    warnings = []
    
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()
    
    # 检查1：frontmatter 完整性
    required_fields = ['title:', 'created:', 'updated:', 'tags:', 'source:', 'enriched:']
    frontmatter_match = re.search(r'^---\n(.*?)\n---', content, re.DOTALL)
    if frontmatter_match:
        fm = frontmatter_match.group(1)
        for field in required_fields:
            if field not in fm:
                errors.append(f"⚠️ frontmatter 缺少: {field}")
    
    # 检查2：wikilinks 孤立检测
    wikilinks = re.findall(r'\[\[([^\]|]+?)(?:\|[^\]]+)?\]]', content)
    # 注：实际使用时需要验证文件是否存在
    
    # 检查3：截图引用检测
    if 'screenshots: true' in fm:
        screenshot_refs = re.findall(r'!\[.*?\]\(([^)]+\.jpg)\)', content)
        if not screenshot_refs:
            warnings.append("⚠️ 标记了 screenshots:true 但笔记里没有实际截图引用")
    
    # 检查4：空章节检测
    sections = re.findall(r'## (.*?)\n(.*?)(?=## |---|\Z)', content, re.DOTALL)
    for title, body in sections:
        if body.strip() == "" or body.strip() == "...":
            errors.append(f"⚠️ 空章节: {title.strip()}")
    
    # 检查5："仍存在的疑问" 非空验证
    questions_section = re.search(r'## ❓ 仍存在的疑问\s*\n\s*\|.*?\||\s*[^\|]', content)
    if not questions_section:
        warnings.append("⚠️ '仍存在的疑问' 部分为空，建议填入真实问题")
    
    return errors, warnings

if __name__ == "__main__":
    errors, warnings = check_note(sys.argv[1])
    if errors:
        print("❌ 错误:")
        for e in errors: print(f"  {e}")
    if warnings:
        print("⚠️ 警告:")
        for w in warnings: print(f"  {w}")
    if not errors and not warnings:
        print("✅ 笔记检查通过")
```

---

## 📋 截图采集记录

```bash
# 本次处理使用的命令（供复现）
# 视频下载
yt-dlp -f "best[height<=720]" -o "/tmp/enrich_video.mp4" "VIDEO_URL"

# 截图列表（共N张）
# 00:02:34 — 概念图示
# 00:05:18 — 茅台财务数据
# 00:08:45 — 商业模式对比
# 00:11:20 — 核心结论
# ...
```

---

## Supported Platforms (as of 2026-05-30)

| Platform | Method | Conditions |
|---|---|---|
| YouTube | `yt-dlp` + HTTP proxy | Need Chrome cookies + Clash HTTP proxy on port 7890 |
| Bilibili | `yt-dlp --cookies-from-browser chrome` | Chrome logged in; `--no-playlist` to avoid downloading series |
| 抖音 | Camoufox + browser_console extract | No login needed; video+audio separate, need FFmpeg merge |

### ⚠️ Output Style: 公众号文章 vs 技术文档

**The trap:** Draft content often reads like technical documentation — dense paragraphs, formal tone, "AI assistant" voice. When the output is for a public WeChat account (公众号), readers bounce immediately.

**卡兹克 style reference** (the benchmark for Chinese tech 公众号):
- 1-3 sentences per paragraph (never dense walls of text)
- Conversational flow: "然后…于是…结果…" instead of "但是…然而…因此…"
- First-person throughout, with emotional color: "万万没想到", "长长的松了一口气", "伤透了脑筋"
- Short sections (2-4 sentences each), not long treatise paragraphs
- Ending with: "以上，既然看到这里了，随手点个赞、在看、转发三连吧。我们下次再见。"
- NO bullet-pointdense lists; use plain prose or 1-2 item lists
- NO block quotes (`>`) for prose — only for actual quotes
- NO "// comments" style — writes like a person talking to a friend

**When generating 公众号 content from this skill:**
1. Break every paragraph >3 sentences into shorter units
2. Replace formal transitions with conversational ones
3. Add first-person emotional color ("终于成功了" not "successfully completed")
4. Strip all markdown `>` block quotes from prose sections
5. End with the reader engagement call-to-action (三连 + 再见)

See `references/kazhig-style-guide.md` for full style analysis.

> **Note**: bilibili cookies 导出：用 EditThisCookie 扩展导出 Netscape 格式，或 Chrome DevTools → Application → Cookies → bilibili.com → Export JSON 后转 Netscape 格式。**不需要手动维护 cookies 文件**，直接用 `--cookies-from-browser chrome` 自动从浏览器提取（1213个cookies实时读取）。

> **YouTube 国内 IP 问题**：国内 IP 直接访问 YouTube 被拦截（"Sign in to confirm you're not a bot"），必须走 HTTP 代理（socks5://127.0.0.1:7891 不兼容，需用 http://127.0.0.1:7890）。

> **抖音 DRM**：作者设置了"不允许下载"时所有自动化方案失效，只能让用户直接在 App 里保存本地再分享文件。

## Support Files

- `references/video-download-platforms.md` — 三平台下载方案对比（YouTube/B站/抖音），含实测命令和踩坑记录
- `references/self_check.py` — 笔记质量自检脚本（frontmatter/wikilinks/空章节检查）
- 截图质量受限

**导出方法（任选一种）：**

**方法 1：Chrome DevTools（推荐）**
1. 打开 Chrome，登录 bilibili.com
2. 按 F12 → Application → Storage → Cookies → bilibili.com
3. 右键表格 → Export as JSON
4. 用脚本转成 Netscape 格式：

```python
# save as scripts/cookies_to_netscape.py
import json, sys
cookies = json.load(open('/path/to/cookies.json'))
print("# Netscape HTTP Cookie File")
for c in cookies:
    print(f"{c['domain']}\tTRUE\t{c['path']}\t"
          f"{'TRUE' if c['secure'] else 'FALSE'}\t"
          f"{c['expires']}\t{c['name']}\t{c['value']}")
```

**方法 2：浏览器扩展**
1. 安装 "Get cookies.txt LOCALLY" (Chrome 扩展)
2. 登录 bilibili.com
3. 点击扩展 → Export → 保存为 `~/cookies_bilibili.txt`

**验证：**
```bash
yt-dlp --cookies ~/cookies_bilibili.txt \
  --list-subs "https://www.bilibili.com/video/BV_ID"
# 应该显示字幕语言列表，而不是"登录可享"
```

### 抖音支持状态

抖音有 DRM 保护，yt-dlp 和浏览器都无法自动下载。

**可行方案：**
1. 在抖音 App 里下载视频 → 隔空投送/Airdrop 到 Mac → 扔给我处理
2. 用户直接发送视频文件给我（MEDIA: 路径）
3. 或者告诉我视频的具体内容描述，我帮你基于描述整理笔记

### YouTube 支持状态

需要先安装 youtube-transcript-api：
```bash
pip install youtube-transcript-api
```

之后 YouTube 视频可以直接发链接给我处理。