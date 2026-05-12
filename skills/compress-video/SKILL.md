---
name: compress-video
description: Compress and re-encode a video file to a smaller MP4 (H.264 + AAC) using ffmpeg. Use when the user asks to compress, shrink, reduce, downsize, or "make smaller" a video; convert .mov / .mkv / .webm / .avi to .mp4; prepare an iOS Simulator, Android Emulator, or React Native screen recording for a GitHub PR (10 MB upload limit); or share a video via Signal, Slack, or email. The user may give an explicit file path or gesture at a location — e.g. "the latest video in Downloads", "my most recent screen recording", "the latest .mov in ~/Movies".
---

# Compress Video

Re-encodes a video file with ffmpeg using defaults tuned for sharing: H.264 video at CRF 28, AAC audio at 128 kbps, and `+faststart` so the file plays before fully downloading.

## Prerequisites

`ffmpeg` must be on `PATH`. If `ffmpeg -version` fails, stop and tell the user to install it (e.g. `brew install ffmpeg` on macOS, `apt install ffmpeg` on Debian/Ubuntu).

## Steps

1. Resolve the input path from the user's message:
   - **Explicit path** — use it directly.
   - **Location reference** (e.g. "the latest video in Downloads", "my most recent recording", "the latest .mov in ~/Movies") — find the most recently modified video file in that directory and confirm with the user before encoding:

     ```bash
     find <dir> -maxdepth 1 -type f \( -iname '*.mov' -o -iname '*.mp4' -o -iname '*.mkv' -o -iname '*.webm' -o -iname '*.avi' \) -print0 | xargs -0 ls -t | head -1
     ```

     One-liner confirmation: "Found `clip.mov`, ~42 MB, modified 5 min ago — compress?"
   - **Still ambiguous** — ask.
2. Verify the file exists. Capture its size (`stat`/`ls -l`) before encoding so you can report the delta.
3. Compute the output path: same directory and basename as the input, suffixed with `-compressed.mp4`. Example: `/Users/me/clip.mov` → `/Users/me/clip-compressed.mp4`.
4. Run, quoting both paths so spaces and special characters survive:

   ```bash
   ffmpeg -i "<input>" -c:v libx264 -preset slow -crf 28 -c:a aac -b:a 128k -movflags +faststart "<output>"
   ```

5. After encoding finishes, report:
   - Original size
   - Compressed size
   - Compression ratio (e.g. "62% smaller" or "2.6× reduction")
   - Output path

## Tuning

- **`-crf 28`** is the quality dial. Lower = better quality + bigger file. `18–23` is visually lossless; `28` is the sharing sweet spot. Drop to `23–25` if the user complains about quality.
- **GitHub PR attachments cap at 10 MB.** If the compressed output is still over that — typical for long iOS Simulator recordings — bump `-crf` toward `30–32` and re-encode, or switch to two-pass with an explicit target bitrate (see below). Always re-check the output size before reporting back.
- **`-preset slow`** trades encode time for compression efficiency. Swap to `medium` or `faster` if the user needs a quick turnaround and tolerates a larger file.
- For a **target file size** (e.g. "under 100 MB"), switch to two-pass encoding: compute `target_video_bitrate = (target_bytes * 8 / duration_seconds) − audio_bitrate`, then run with `-b:v <bitrate> -pass 1` and `-b:v <bitrate> -pass 2`.

## Skip this skill when

- The user wants to **trim, clip, or splice** the video — that is different ffmpeg flags (`-ss`, `-to`).
- The output should be **a non-MP4 format** (WebM, GIF, animated WebP) — this skill is MP4-only.
- The user wants to **inspect** metadata or codec info — use `ffprobe` directly instead.
- The user wants to **extract audio** or **strip audio** — different flag set.
