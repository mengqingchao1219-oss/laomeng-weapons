---
name: creator-clone-lab
description: End-to-end creator benchmark distillation and AI creator clone workflow. Use when the user asks to 蒸馏博主, analyze a creator or benchmark video/image post, 抓取/下载对标视频或图文, analyze 小红书/抖音/快手/B站 benchmark content, extract how a creator chooses topics/thinks/expresses, build a creator AI 分身, import benchmark references, or run the content loop from benchmark research to scripts, scoring, prediction, publishing records, retrospectives, and rubric calibration.
---

# Creator Clone Lab

## Purpose

Turn a creator, account, or group of benchmark videos into a reusable content operating system:

```text
collect source material -> distill creator -> extract thinking patterns -> build AI creator clone -> generate topics/scripts -> score/predict/retro/calibrate
```

Produce concrete artifacts, not only commentary.

## Core Principles

- Prefer original videos, images, frames, audio, captions, metadata, comments, and performance data over title-only guessing.
- Distill the creator's decision system, not just surface wording.
- Separate high-like, high-share, high-comment, high-save, and weak samples before summarizing patterns.
- Treat the AI creator clone as a ruleset with taste, boundaries, and self-checks.
- Preserve benchmark sources and connect them to the later loop: script, score, prediction, publishing record, retrospective, and rubric update.
- Never expose cookies, secrets, or login tokens. Delete temporary browser profiles/cookie files after use unless the user explicitly asks to keep them.

## Workflow

### 0. Check And Install Local Tools

Before any capture or distillation, check whether the local machine can process video, audio, screenshots, subtitles, and browser sessions.

Run the bundled checker from this skill folder:

```bash
python scripts/check_install_media_tools.py
```

If anything is missing, install automatically:

```bash
python scripts/check_install_media_tools.py --install
```

If `ffmpeg` or `ffprobe` is missing, also allow system-tool installation:

```bash
python scripts/check_install_media_tools.py --install --install-system
```

Required capabilities:

- `yt-dlp`: resolve and download public video pages when possible.
- `websocket-client`: attach to browser debug sessions when login is already open.
- `ffmpeg` and `ffprobe`: inspect media, extract audio, cut frames.
- `faster-whisper`: ASR speech-to-text for no-subtitle videos with voiceover/dialogue.
- `rapidocr-onnxruntime`, `opencv-python-headless`, `pillow`: OCR and frame processing for burned-in subtitles and screenshots.
- Optional `GROQ_API_KEY`: API fallback for speech-to-text when local ASR cannot be installed.

Rules:

- Always run this check before first use on a new machine.
- On Windows Chinese locale, set UTF-8 mode before running validation or scripts if text decoding fails:

```bash
set PYTHONUTF8=1
```

- If a required Python package is missing, run the checker with `--install` before capture.
- Run the checker, capture scripts, ASR, and OCR with the same Python or virtual environment. If `python scripts/check_install_media_tools.py` reports missing packages but another environment already works, switch to that environment before installing duplicates.
- If `ffmpeg`/`ffprobe` is missing, run with `--install-system` or give the user the exact install command printed by the script.
- If ASR/OCR is missing, do not claim full understanding of no-subtitle videos. Mark the sample as partial until the tools are installed.
- If local ASR cannot be installed, use the bundled API fallback:

```bash
python scripts/transcribe_audio.py input.mp4 --provider groq --output transcript.txt --language zh
```

- The API fallback requires `GROQ_API_KEY` in the environment. If it is not set, ask the user to create a Groq API key and set it before retrying.
- For Chinese talking-head or creator-explainer videos, Groq `whisper-large-v3-turbo` is often more accurate than local `faster-whisper small` on creator terms. If accuracy matters more than offline processing, use `--provider groq`.
- Local `faster-whisper small` may mishear specialized Chinese creator terms such as `蒸馏` as similar-sounding words. If this happens, retry with `--local-model medium` or use Groq.
- Tell the user the first ASR run may download a Whisper model and take extra time.

### 1. Clarify The Benchmark Target

Identify what the user provided:

- A single video link.
- A single image-text post link.
- A creator homepage/account link.
- Multiple benchmark links.
- A local video/image/data folder.
- A desired creator archetype without links.

If the link is a homepage, do not treat it as one video. Extract the account name, bio, visible video count, and available works.

### 2. Choose Platform Strategy

Support benchmark capture across platforms, but treat each platform as a different adapter. Do not promise full extraction until the current link/account has been tested.

Platform matrix:

```text
Douyin: video-first; public short-link first; logged-in browser fallback; ASR/OCR required for no-subtitle videos.
Xiaohongshu: image-text and video; capture title/body/tags/images/comments/likes/collects; OCR and image-layout analysis are core, not optional.
Kuaishou: video-first; public link first; logged-in browser fallback; ASR/OCR for spoken or burned-in text.
Bilibili: video-first plus long-form metadata; public pages often work; capture title/desc/tags/danmaku/comments when available; ASR for narration.
Local folders: treat videos/images/screenshots as source material and build metadata manually from filenames or user notes.
Other platforms: use the same generic adapter: public page -> existing logged-in browser -> user-assisted login -> screenshots/OCR/manual fallback.
```

For every platform:

1. Try public/no-login extraction first.
2. If blocked, check for an already-open logged-in browser session.
3. If the existing session is logged in, reuse it without asking the user to log in again.
4. If no logged-in session exists, ask the user to log in or pass verification.
5. If media download is blocked, use screenshots, visible text, OCR, ASR, and manually captured metrics, then label the source as partial.

### 3. Capture Source Material

For web platforms:

1. Resolve short links and identify whether each link points to a video or profile.
2. Try public/no-login access first with page metadata, platform extractors, and direct media/profile endpoints.
3. If public access fails, check whether the user already has a browser open with an active login session.
4. If the user is already logged in, reuse that session and continue capture. Do not ask them to log in again.
5. If no usable logged-in session exists, open a browser and ask the user to log in or pass verification.
6. Create an isolated temporary browser profile only when the existing session cannot be used or isolation is safer.

Capture as much as possible:

- Creator/account metadata.
- Video IDs and URLs.
- Titles, descriptions, hashtags.
- Publish time if available.
- Duration.
- Like/comment/share/collect/play counts if available.
- Original video files where feasible.
- Original image files where feasible.
- Keyframe contact sheets.
- Image contact sheets for image-text posts.
- Audio transcript when the video has voiceover or dialogue.
- On-screen text via OCR when subtitles/captions are burned into the video.
- OCR text from images, screenshots, cover images, and carousel slides.
- Visible top comments and comment counts when available.
- Visual scene notes from frames when there is no usable text or speech.

For content understanding, use this fallback stack:

```text
metadata/title/description
-> OCR on frames for on-screen subtitles
-> local ASR/audio transcription for voiceover/dialogue
-> API ASR fallback when local ASR cannot be installed
-> visual frame understanding for silent/no-text scenes
```

Never mark a sample as fully understood when only title/description was captured. Use `metadata-only` or `partial` until transcript/OCR/frame notes are available.

When working with Douyin:

- First try no-login extraction with short-link resolution, video page metadata, and platform extractors.
- If no-login extraction cannot list videos or download media, check for an already-open Douyin browser session and reuse it when logged in.
- If an existing Douyin browser is open but not logged in, tell the user exactly what is needed: "请在当前抖音窗口登录/过验证，完成后回复登录好了。"
- Prefer authenticated browser API responses for profile video lists.
- Store source videos under `samples/<creator>/videos/`.
- Store contact sheets under `samples/<creator>/frames/`.
- Store transcripts under `samples/<creator>/transcripts/`.
- Store OCR and scene notes under `samples/<creator>/notes/`.
- Store structured metadata as `samples/<creator>_items.json`.
- Remove temporary login profiles/cookies after capture. Do not remove the user's normal browser profile.

When working with Xiaohongshu:

- Support both notes and videos.
- For profile links, first try the bundled public SSR extractor:

```bash
python scripts/extract_xhs_profile.py "<xhs profile url>" --out samples/<creator> --download-covers
```

- For image-text notes, capture title, body text, hashtags, author info, publish time if visible, likes, collects, comments, and every image in the carousel when feasible.
- Run OCR on every image and screenshot. Many Xiaohongshu posts carry the real hook, argument, list, or selling point inside the image, not only in the body text.
- Analyze cover image separately: cover title, visual hierarchy, contrast, face/product presence, keyword density, and curiosity gap.
- For video notes, use the same video stack as Douyin: download if possible, extract frames, OCR burned-in text, and transcribe speech.
- For Xiaohongshu profile pages, public SSR may expose video cards, titles, like counts, and cover images while hiding `note_id` and video source URLs. Treat these as benchmark candidates, not downloadable videos.
- To download a Xiaohongshu video, prefer a single video note/detail link or an already-open logged-in browser session. Inspect the detail page/network responses for video URLs, then download with browser `User-Agent`/`Referer`.
- If a logged-in Chrome session is needed, ask the user to log in, then reuse that browser through Chrome remote debugging instead of requesting cookies or tokens. Example launch command:

```bash
chrome.exe --remote-debugging-port=9222 --remote-allow-origins=* --user-data-dir=%TEMP%\\xhs-debug https://www.xiaohongshu.com/explore
```

- After the user logs in and provides a single note/detail URL, run:

```bash
python scripts/capture_xhs_video_from_debug_browser.py "<xhs note url>" --out samples/<creator>/<note_id>
```

- The script attaches to `http://127.0.0.1:9222`, watches network responses for Xiaohongshu video sources such as `sns-video`/`video/mp4`, saves `network_video_urls.json`, and downloads the first captured MP4 immediately. Signed video URLs may expire, so do not store them as permanent source links.
- If only a profile link is available and video `note_id`/source URL is hidden, download covers, run OCR, rank candidates, and ask the user for the target video note link or login help before claiming video download support.
- If download is blocked, take screenshots of the cover, carousel pages, body, and comments. Store the result as partial instead of failing.
- Store source images under `samples/<creator>/images/`.
- Store screenshots under `samples/<creator>/screenshots/`.
- Store OCR/image notes under `samples/<creator>/notes/`.
- Store structured metadata as `samples/<creator>_items.json`.
- Public SSR extraction may expose the first page of profile cards, titles, cover images, types, like counts, and pagination cursor, but detail pages, comments, full carousel media, or exact account-level metrics may still require a logged-in browser session.

When working with Kuaishou:

- First try no-login extraction with the public URL.
- If blocked or incomplete, reuse an already-open logged-in browser session before asking for login.
- Capture video, cover, title/caption, tags, visible metrics, comments, frames, OCR, and ASR transcript.
- If the page only exposes playback screenshots, mark download as no and understanding as partial unless OCR/ASR evidence is sufficient.

When working with Bilibili:

- Prefer public page extraction first because many video pages expose title, desc, tags, duration, and playback metadata without login.
- For a single video or `b23.tv` short link, first try the bundled extractor:

```bash
python scripts/extract_bili_video.py "<bilibili or b23 url>" --out samples/<video_name> --download
```

- Capture uploader info, title, description, tags, publish time, duration, views, likes, coins, favorites, shares, comments, and danmaku when available.
- For long videos, extract representative segments or keyframes instead of transcribing the entire video when the user only needs style distillation.
- Use ASR for narration and OCR for on-screen subtitles/slides.
- If `yt-dlp` returns HTTP 412 on Bilibili, do not stop. Resolve the `bvid`, call `x/web-interface/view` for metadata, then call `x/player/playurl` for DASH video/audio URLs. Download video and audio with Bilibili `Referer`/browser `User-Agent`, then merge with ffmpeg.
- Bilibili comment APIs may require signed WBI parameters or login. If comments return `访问权限不足`, save the available metadata and mark comments as not captured.

### 4. Mark Source Understanding Status

For every captured sample, record:

```text
platform: douyin / xiaohongshu / kuaishou / bilibili / local / other
content type: video / image-text / mixed
video downloaded: yes/no
images downloaded: yes/no
frames extracted: yes/no
OCR subtitles: yes/no
ASR transcript: yes/no
comments captured: yes/no
understanding level: full / partial / metadata-only
```

Use these labels:

- `full`: media, metadata, and either transcript/OCR/image notes/comments or sufficient visual notes are available.
- `partial`: some media/frames/images/metadata are available, but key speech, image text, comments, or on-screen text is missing.
- `metadata-only`: only title, description, hashtags, or performance data are available.

### 5. Segment Performance

Create a performance table before distillation.

Classify samples by:

- Highest likes: strongest emotion or identity resonance.
- Highest shares: strongest forwarding reason.
- Highest comments: strongest debate, confession, or participation hook.
- Highest saves/collects: strongest utility, template, or rewatch value.
- Weak samples: boundaries of what the audience does not reward.

Do not average everything too early. Preserve the difference between each metric.

### 6. Distill The Creator

Extract five layers.

**Layer 1: Positioning**

- What the creator is really selling.
- The recurring promise to the audience.
- The hidden genre, such as "pet diary" actually being "pet personality sitcom".

**Layer 2: Topic Selection**

- Repeating topic buckets.
- Trigger events: holidays, daily routines, social identities, conflict scenes, audience comments.
- What makes a topic "theirs" rather than generic.

**Layer 3: Thinking Pattern**

- What assumption the creator makes about the audience.
- What tension they look for.
- How they decide which detail deserves a video.
- How they balance novelty and familiar format.

**Layer 4: Expression Pattern**

- Opening hook.
- Scene order.
- Shot types.
- Subtitle/voiceover/silence ratio.
- Whether key information is carried by title, body text, image text, cover, on-screen subtitles, spoken audio, comments, or pure visuals.
- For image-text posts: cover promise, carousel rhythm, headline density, text-image relationship, final slide CTA, and comment-bait design.
- Humor, emotion, irony, suspense, or tenderness mechanics.
- Ending package: punchline, reveal, emotional soft landing, call-back, or action cue.

**Layer 5: Transferable Template**

Turn the pattern into reusable formulas:

```text
Formula name
When to use
Input material needed
Beat-by-beat structure
Caption/subtitle/voice style
Expected metric strength
Risks
```

### 7. Build The AI Creator Clone

Create a reusable creator clone spec that answers:

- Who is this clone?
- What topics does it accept or reject?
- How does it choose an angle?
- How does it open?
- How does it write scenes?
- How does it use subtitles, narration, and silence?
- How does it use covers, carousel slides, body text, comments, and visual proof?
- What does it never do?
- How does it self-check whether a draft fits the creator?

Recommended artifact: `creator_clone.md` or a section in `benchmark.md`.

Minimum clone template:

```markdown
# Creator Clone: <name>

## Taste
## Audience Assumption
## Topic Selection Rules
## Structure Rules
## Expression Rules
## Visual Rules
## Image-Text Rules
## Caption/Subtitle Voice
## Ending Rules
## Anti-Patterns
## Self-Check Rubric
```

### 8. Connect To The Local Content Loop

If the workspace has local content-calibration files:

1. Write or update `benchmark.md` with the distillation.
2. Save samples and metadata under `samples/`.
3. Update the local state file if present:
   - `benchmark_status: "imported"`
   - `benchmark_name: "<creator or benchmark set>"`
   - `benchmark_sample_count: <n>`
4. Do not treat benchmark videos as the user's calibration samples. Calibration samples come from the user's own published videos and retrospectives.
5. Use the benchmark to generate candidate topics, then scripts.

### 9. Generate From The Clone

When asked for ideas or scripts, use the clone rules.

For each generated idea, include:

- The benchmark formula it uses.
- Why this creator would choose it.
- Likely strength: likes, shares, comments, saves, or watch time.
- Production requirements.

For scripts, write scene beats rather than generic essay text:

```text
镜头 / 画面
字幕 or 旁白
角色动作
节奏 purpose
```

For image-text posts, write slide beats:

```text
封面 / hook
第 2-N 页 / 信息推进
每页主标题
画面元素
正文补充
评论区引导
```

### 10. Score, Predict, Publish, Retro, Calibrate

Use this loop:

```text
score draft -> blind prediction -> publish -> T+3/T+7 data -> retro -> rubric/pattern update
```

During retrospectives, compare:

- Which cloned rule worked.
- Which rule failed.
- Whether the topic, opening, expression, or platform fit caused the result.
- What should change in the clone or benchmark.

## Output Standards

For a full creator distillation, produce:

1. Source inventory table.
2. Performance segmentation.
3. Creator positioning.
4. Topic buckets.
5. Thinking pattern.
6. Expression/visual pattern.
7. Transferable templates.
8. AI creator clone rules.
9. Next 5-20 candidate ideas.
10. Files updated, if working in a local project.

For a quick distillation, produce:

1. What this creator is really doing.
2. 3 reusable formulas.
3. 5 candidate ideas in that style.

## Failure Modes

- If video download fails, use page metadata and screenshots only after stating the limitation.
- If a video has no subtitles, do not stop at title/description. Extract audio for ASR if there is speech, and use frame/OCR analysis for visual text.
- If ASR/OCR is unavailable or fails, explicitly say the sample is partial and avoid overconfident claims about script structure or spoken hooks.
- If Groq transcription returns HTTP 403 with `error code: 1010`, first retry with the bundled `transcribe_audio.py`; it sends `User-Agent` and `Accept` headers that some gateway paths require. If it still fails, verify the key with curl, then check network/VPN/proxy restrictions.
- If Groq says the file is too large, compress or extract audio with ffmpeg before retrying. Groq documents a 25 MB direct upload limit for the free tier.
- If local ASR completes but key terms look wrong, compare against Groq or a larger local model before using the transcript for creator-pattern distillation.
- If public capture fails, first check for an already-open logged-in browser session. Ask the user to log in only when no usable logged-in session exists.
- If Xiaohongshu image-text media cannot be downloaded, screenshot every slide and run OCR; do not treat body text alone as full understanding.
- If Xiaohongshu video cards come from a profile page with blank `note_id`, do not pretend the source video was captured. Mark `video downloaded: no`, keep the cover/OCR/title/metrics, and request a single note link or logged-in browser session.
- If a platform hides metrics or comments, record the visible fields and label missing fields explicitly.
- If only one sample is available, label the result as provisional and avoid overfitting.
- If the creator has mixed formats, separate formats before deriving rules.
- If the user asks to copy a creator exactly, reframe as learning the structure and creating an original adjacent style.
