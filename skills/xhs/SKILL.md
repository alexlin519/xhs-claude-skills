---
name: xhs
description: 提取小红书帖子内容（文字、图片 OCR、视频转录），生成"摘要 + 原文"的 Obsidian 笔记
user-invocable: true
argument-hint: <小红书链接>
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

用户希望提取小红书帖子内容。请按以下步骤处理：

## 常量定义
- Cookies 文件: `~/cookies.json`（从 Chrome 导出的小红书 cookies）
- Obsidian 保存目录: `/Users/alexlin/Library/Mobile Documents/iCloud~md~obsidian/Documents/龍行天下/xhs`
- Whisper 模型: `mlx-community/whisper-large-v3-turbo`

## 输入
用户提供的小红书链接: $ARGUMENTS

## 提取流程

### 步骤 0：检查 Cookies
1. 检查 `~/cookies.json` 是否存在
2. 如果不存在，告知用户需要从 Chrome 导出 cookies：
   - 在 Chrome 打开 xiaohongshu.com 并确认已登录
   - 打开 DevTools Console，运行以下代码将 cookies 复制到剪贴板：
   ```javascript
   copy(JSON.stringify(document.cookie.split('; ').map(c => {
     const [name, ...rest] = c.split('=');
     return { name, value: rest.join('='), domain: '.xiaohongshu.com', path: '/',
       expires: Date.now()/1000 + 86400*30, size: name.length + rest.join('=').length,
       httpOnly: false, secure: false, session: false, priority: 'Medium',
       sameParty: false, sourceScheme: 'Secure', sourcePort: 443 };
   })))
   ```
   - 将剪贴板内容保存到 `~/cookies.json`
   - 然后终止流程，等用户完成后重新运行

### 步骤 1：解析链接
从 URL 中提取帖子 ID（24 位十六进制字符串）和 xsec_token 参数。

### 步骤 2：获取帖子内容
使用 Python 脚本，通过 Cookies 请求帖子页面 HTML，从 `window.__INITIAL_STATE__` 解析全部帖子数据：

```python
import json, urllib.request, ssl, re

with open('<Cookies 文件>') as f:
    cookies = json.load(f)
cookie_str = '; '.join(f"{c['name']}={c['value']}" for c in cookies)

ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE

req = urllib.request.Request('<帖子URL>')
req.add_header('Cookie', cookie_str)
req.add_header('User-Agent', 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36')

resp = urllib.request.urlopen(req, timeout=15, context=ctx)
html = resp.read().decode('utf-8', errors='ignore')

m = re.search(r'window\.__INITIAL_STATE__\s*=\s*(\{.+?\})\s*</script>', html, re.DOTALL)
raw = m.group(1).replace('undefined', 'null')
data = json.loads(raw)

# 帖子数据在: data['note']['noteDetailMap'][<key>]['note']
# 包含: title, desc, type, time, user, imageList, video, interactInfo, ipLocation
```

如果请求失败（被重定向到 404/错误页），说明 cookies 过期，提示用户按步骤 0 重新导出。

### 步骤 3：提取原文（图文 OCR / 视频转录）
根据帖子 type 分两种情况，目标都是得到帖子的**原文文字**（既不嵌入也不保存图片）：

#### 情况 A：图文帖子（type 为 normal / image）
小红书图文帖子的正文往往写在图片里（截图、手写笔记、长图文），需要 OCR 把文字提取出来：

1. 从 `note['imageList']` 中按顺序取每张图片的 `urlDefault`
2. 下载到临时文件（仅临时，不进 Obsidian）：
```bash
curl -L -o /tmp/xhs_{post_id}_{n}.jpg -H "Referer: https://www.xiaohongshu.com/" <图片URL>
```
3. **用 Read 工具逐张打开临时图片**，让 Claude 用视觉直接识别图中文字（无需安装任何 OCR 依赖）
4. 把 `desc`（帖子描述）与各图 OCR 文本**按图片顺序合并**为完整原文；只做轻度清理（去掉 `#xxx[话题]#` 标记、明显的水印/广告语），不改写、不缩写
5. 清理临时文件：
```bash
rm -f /tmp/xhs_{post_id}_*.jpg
```

#### 情况 B：视频帖子（type 为 video）
保留 whisper 语音转录作为原文：

##### 3b-1. 提取视频 URL
从步骤 2 获取的数据中解析视频流：
```
note['video']['media']['stream'] -> 按 h264 > h265 > av1 优先级取第一个的 masterUrl
```

##### 3b-2. 下载视频并提取音频
```bash
curl -L -o /tmp/xhs_{post_id}.mp4 -H "Referer: https://www.xiaohongshu.com/" <视频URL>
ffmpeg -y -i /tmp/xhs_{post_id}.mp4 -vn -acodec pcm_s16le -ar 16000 -ac 1 /tmp/xhs_{post_id}.wav
```

##### 3b-3. 语音转录
```python
import mlx_whisper
result = mlx_whisper.transcribe("/tmp/xhs_{post_id}.wav",
    path_or_hf_repo="mlx-community/whisper-large-v3-turbo", language="zh", verbose=False)
```

##### 3b-4. 清理转录文本
- 去除尾部重复字符（背景音乐噪音）
- 按语义断句，添加标点和段落
- 如有步骤/要点结构，用 Markdown 格式化

##### 3b-5. 清理临时文件
```bash
rm -f /tmp/xhs_{post_id}.mp4 /tmp/xhs_{post_id}.wav
```

### 步骤 4：整理输出并保存
将内容整理为 Markdown 文件，保存到 `<Obsidian 保存目录>/{YYYY-MM-DD} {短标题}.md`。
- 文件名格式：`{发布日期} {短标题}.md`，短标题不超过15个字，是核心洞察的极简概括
- 日期前缀确保按时间排序
- 不创建子目录，所有帖子 md 直接放在 xhs 文件夹下
- **不保存任何图片/视频文件**，也不嵌入图片——笔记只含文字（摘要 + 原文）

**写作风格：Peter Thiel 式——直接、反直觉、一句话给判断。笔记是决策工具，不是知识库。用户扫一眼就能决定：深挖还是跳过。**

文件结构（**无 YAML frontmatter**）：

```markdown
# 一句话核心洞察（反直觉的判断，不是描述性标题）

核心论点，2-3句话。直接给出"大多数人觉得X，但其实Y"的判断。
不废话，不铺垫，像 Thiel 在董事会上说话。

**与我的关联：** 一句话。读取用户的 memory（~/.claude/projects/*/memory/ 下的
user 和 project 类型记忆）了解用户背景、研究方向和当前工作，据此说清楚
这个内容跟用户有什么关系。如果 memory 不可用，从通用的个人发展/工具/方法论角度切入。

**值得深挖吗：** 是/否。一句话理由。

## 摘要
对这篇帖子 / 视频的 3-5 句话摘要：讲了什么、核心方法或结论是什么。
这是判断"要不要深挖"的依据，比上面的一句话洞察更完整，但仍然精炼。

> [!quote]- 原文
> 帖子的**原始文字**（折叠状态，点开才看到），用于存档和精读：
> - 图文帖子：`desc` + 各图 OCR 文本，按图片顺序合并
> - 视频帖子：整理后的语音转录全文
> - 只做轻度清理（去 `#xxx[话题]#` 标记、断句加标点），**不改写、不缩写**，保留原意

> [!info]- 笔记属性
> - **来源**: 小红书 · 作者名
> - **帖子ID**: xxx
> - **链接**: 原始链接
> - **日期**: YYYY-MM-DD
> - **类型**: image/video
> - **互动**: N赞 / N收藏 / N评论
> - **标签**: 标签1, 标签2, ...
```

关键约束：
- 折叠区域外的可见内容（洞察 + 关联 + 值得深挖 + 摘要）保持精炼
- 标题必须是洞察/判断，不是"XX帖子的总结"
- 笔记**纯文字**：不嵌入、不下载图片；图文帖子的内容通过 OCR 进入「原文」折叠区
