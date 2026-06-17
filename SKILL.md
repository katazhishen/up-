---
name: bilibili-recognition
description: >
  搜索B站UP主视频 → 下载音频 → faster-whisper转文字 → AI总结 → 生成.docx。
  当用户提到"B站"、"UP主"、"视频总结"、"视频下载"、"转写"、
  "视频转文档"、"视频内容整理"、"B站视频识别"或任何B站视频分析需求时，
  务必使用此技能，即使只作为参考流程。
---

# B站UP主视频 → 音频转写 → AI总结 → Docx

## 流程总览

```
用户提供UP主名称/空间URL
        ↓
Chrome DevTools MCP 抓取视频列表（标题/BV号/日期/时长）
        ↓
询问: "获取最近几个视频?"（建议 ≤5个，太多转写耗时很长）
        ↓
浏览器DASH提取音频直链 → curl下载 → ffmpeg转m4a
        ↓
faster-whisper 转写音频 → .json (带时间戳)
        ↓
Claude 逐视频结构化总结 (一句话概述 + 核心观点 + 关键信息 + 受众价值)
        ↓
python-docx 生成文档 → D:\B站下载\<UP主>_总结.docx
```

## 依赖检查

首次使用时确认环境:

```bash
pip install yt-dlp
```

已内置: `faster-whisper` `python-docx` `ffmpeg`（缺了会自动提醒安装）。

---

## 步骤详解

### 1. 获取视频列表

用 Chrome DevTools MCP 导航到 `https://space.bilibili.com/<uid>/video`，执行 JS 抓取:

```javascript
() => {
  const videos = [], seen = new Set();
  document.querySelectorAll('a[href*="/video/BV"]').forEach(link => {
    const bv = link.href.match(/BV[\w]+/)?.[0];
    if (!bv || seen.has(bv)) return;
    const text = link.textContent?.trim() || '';
    if (/^[\d.]+万/.test(text) || text.length < 3) return;
    seen.add(bv);
    const card = link.closest('li');
    const dateMatch = card?.textContent?.match(/(\d{2}-\d{2})/);
    const durMatch = card?.textContent?.match(/(\d{1,2}:\d{2}(?::\d{2})?)/);
    videos.push({
      title: text.substring(0, 120), bv,
      date: dateMatch?.[1] || '',
      duration: durMatch?.[1] || '',
      url: 'https://www.bilibili.com/video/' + bv
    });
  });
  return JSON.stringify(videos);
}
```

展示前10条摘要给用户，询问要几个。

### 2. 下载音频

逐个视频页提取DASH音频直链，利用浏览器登录态绕过认证:

**提取音频链接:**
```javascript
() => {
  const dash = window.__playinfo__?.data?.dash;
  if (!dash) return JSON.stringify({error: 'no playinfo'});
  const best = dash.audio.sort((a,b) => b.bandwidth - a.bandwidth)[0];
  return JSON.stringify({url: best.baseUrl, bw: best.bandwidth});
}
```

**下载（URL有时效性，提取后立即执行）:**
```bash
curl -L -o "D:/B站下载/<UP主>/<日期>_<标题>.m4s" \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  -H "Referer: https://www.bilibili.com/" \
  "<提取到的URL>" && \
ffmpeg -i "D:/B站下载/<UP主>/<日期>_<标题>.m4s" \
  -c copy "D:/B站下载/<UP主>/<日期>_<标题>.m4a" -y
```

> 此方式实战验证通过，利用浏览器已登录态直接获取CDN音频链接。

**备选: yt-dlp CLI**（可能因B站WBI签名返回412）:
```bash
python scripts/downloader.py \
  --bv-list video_list.json \
  --output-dir "D:\B站下载" --up-name "<UP主名>"
```

### 3. 转写音频

```bash
python scripts/transcriber.py \
  --input-dir "D:\B站下载\<UP主名>" \
  --output "D:\B站下载\<UP主名>\transcriptions.json" \
  --model "large-v3" --language "zh"
```

- 首次使用 large-v3 需下载约3GB模型（如设代理: `export HTTPS_PROXY=http://127.0.0.1:7897`）
- 网络不通时降级: `--model "medium"` 或 `"tiny"`
- 自动跳过已转写文件（断点续转）
- 输出含逐段时间戳和全文

### 4. 总结凝练

读取 `transcriptions.json` 中每个视频的 `full_text`，用以下模板逐一总结:

```
你是内容总结助手。请对以下B站视频的音频转写内容进行结构化总结。

要求:
1. **一句话概述**: 用一句话说明视频核心内容
2. **核心观点** (3-5条): 最重要的论点，每条不超过50字
3. **关键信息**: 值得记住的数据、事实或金句
4. **受众价值**: 什么人适合看、能获得什么启发

---
视频标题: {title}  发布日期: {date}  时长: {duration}

转写文本:
{full_text}
```

将总结结果整理为 `summaries.json`:

```json
{
  "up_name": "UP主名称",
  "generated_at": "YYYY-MM-DD",
  "videos": [{
    "title": "...", "bv": "BV...", "date": "MM-DD", "duration": "mm:ss",
    "summary": {
      "one_liner": "一句话",
      "key_points": ["观点1", "观点2"],
      "key_info": ["数据/金句"],
      "audience_value": "价值说明"
    },
    "full_transcription": "完整转写原文"
  }]
}
```

### 5. 生成Docx文档

```bash
python scripts/docx_generator.py \
  --input "D:\B站下载\<UP主名>\summaries.json" \
  --output "D:\B站下载\<UP主名>_总结.docx"
```

文档结构: 封面 → 目录 → 每视频(标题/元信息/概述/观点/关键信息/价值) → 附录(完整转写)

---

## 常见问题

| 问题 | 解决 |
|------|------|
| yt-dlp HTTP 412 | 用浏览器DASH直链方式（步骤2主方案） |
| HuggingFace下载模型超时 | 设代理或换小模型 `--model "medium"` |
| large-v3太大下载慢 | 用 `--model "medium"` 平衡速度和精度 |
| 转写结果乱码 | 确保先 `ffmpeg -i in.m4s -c copy out.m4a` 转容器 |

## 所需权限

- `Bash`: 运行脚本、curl下载、pip安装
- `mcp__chrome-devtools`: B站页面抓取+DASH提取
- `Write`: 写入 D:\B站下载\ 和项目目录
- `Read`: 读取转录/总结JSON
