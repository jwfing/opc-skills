---
name: video-subtitles
description: End-to-end workflow for adding subtitles to screen recordings on macOS. Use this skill whenever the user wants to add subtitles or captions to a video, transcribe a screen recording, generate an SRT file from a video, burn subtitles into an MP4, or polish a Screen Studio recording with captions. Also use whenever the user mentions whisper, whisper.cpp, Subtitle Edit, ffmpeg subtitle burning, or any combination of "video + subtitle/caption/transcribe" — even if they don't explicitly mention this skill or all the tools by name.
---

# Video Subtitles Workflow (macOS)

Complete pipeline for taking a screen recording (typically from Screen Studio) and producing a final MP4 with burned-in subtitles. Optimized for short product demos in the 1-10 minute range.

## Toolchain

This workflow uses four tools, each for one specific job:

| Tool | Role |
|---|---|
| Screen Studio | Record source video, export as MP4 |
| ffmpeg (8.x) | Extract audio from video as 16kHz mono WAV |
| whisper-cpp | Transcribe audio to SRT subtitle file |
| Subtitle Edit | Review/edit SRT against video preview and waveform |
| ffmpeg@7 | Burn subtitles into the final MP4 (requires libass, which 8.x lacks) |

**Critical:** Homebrew's ffmpeg 8.x is compiled without libass, so the `subtitles` filter does not exist. Burning subtitles MUST use `ffmpeg@7`. Both versions can be installed in parallel; `ffmpeg@7` is keg-only and won't conflict.

## Prerequisites (one-time setup)

```bash
# Both ffmpeg versions
brew install ffmpeg ffmpeg@7

# Whisper transcription engine
brew install whisper-cpp

# Subtitle editor (GUI)
brew install --cask subtitleedit

# Download the large-v3 model (~3GB, best accuracy)
mkdir -p ~/whisper-models && cd ~/whisper-models
curl -LO https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-large-v3.bin
```

Verify the burning capability is available:

```bash
/opt/homebrew/opt/ffmpeg@7/bin/ffmpeg -filters 2>/dev/null | grep subtitles
# Should print a line containing "subtitles"
```

## Stage 1: Record with Screen Studio

Record the video with Screen Studio, ensuring the **microphone track is captured** (Whisper needs voice audio). Export as MP4.

If the project file (`.screenstudio`) is still available and you only need basic captions, Screen Studio has built-in captions via Whisper — that's the fastest path. This skill covers the case where only the exported MP4 remains, or where you need full control over caption text/styling.

## Stage 2: Extract audio for transcription

Whisper.cpp requires 16kHz mono PCM WAV input — it will not read MP4 directly.

```bash
ffmpeg -i input.mp4 -ar 16000 -ac 1 -c:a pcm_s16le audio.wav
```

**Common typo:** `pcm_s16le` is "16-bit little endian" — lowercase L+E, not the digit 1+E. Writing `pcm_s161e` produces "Unknown encoder" errors.

## Stage 3: Transcribe with whisper.cpp

```bash
whisper-cli -m ~/whisper-models/ggml-large-v3.bin \
  -l en \
  -osrt \
  -t 8 \
  --prompt "Domain context with proper nouns and technical terms specific to this video. List acronyms and product names so the model spells them correctly." \
  audio.wav
```

Output: `audio.wav.srt` in the current directory.

**Key flags:**
- `-l en` — language code (`zh` for Chinese, `auto` to detect)
- `-osrt` — output SRT format (also: `-ovtt`, `-otxt`, `-oj` for JSON)
- `-t 8` — thread count, set to physical core count on Apple Silicon
- `--prompt` — provides context to the model. **This dramatically improves accuracy on technical/proper nouns** (product names, acronyms, domain jargon). Always populate this for technical content.

**Optional speedups (M3/M4):**
- `-fa` — Flash Attention
- `-p 2` — parallel processors for long audio

Rename the output to a cleaner filename for downstream use:

```bash
mv audio.wav.srt subtitle.srt
```

## Stage 4: Review and edit the SRT

Two paths depending on what needs fixing:

### Path A: Text-only fixes (Whisper got words wrong, but timing is fine)

Open in any text editor. SRT is plain text:

```
1
00:00:01,200 --> 00:00:04,500
This is the first subtitle line

2
00:00:04,800 --> 00:00:07,200
This is the second one
```

VS Code with the "Subtitle Edit" extension is fastest for pure text correction.

### Path B: Timing or splitting issues (use Subtitle Edit GUI)

```bash
open -a "Subtitle Edit"
```

Workflow inside Subtitle Edit:

1. `File → Open` → load `subtitle.srt`
2. `Video → Open video file` → load the MP4
3. If waveform doesn't auto-generate: `Video → Generate waveform data`
4. Click any subtitle row → video jumps to that timestamp, waveform highlights the segment
5. Edit text in the bottom edit box (Enter to save the line)
6. Drag waveform region edges to adjust timing visually
7. Cmd+Click multiple rows for multi-select operations
8. Right-click selection → `Merge selected lines` or `Split line`
9. `Cmd+S` to save

**Bulk time shift** (if all subtitles are uniformly off): `Synchronization → Adjust all times` → enter offset (positive = delay, negative = advance).

**Alternative for command-line bulk shift:**
```bash
ffmpeg -itsoffset 0.5 -i subtitle.srt -c copy subtitle_shifted.srt
```

### Quick preview between edits (mpv)

```bash
mpv input.mp4 --sub-file=subtitle.srt
```

mpv hotkeys: `Space` pause, `←/→` ±5s, `,/.` frame step, `z/Z` shift subtitles ±0.1s, `v` toggle subtitles.

mpv is preview-only — it cannot edit. Loop: edit in VS Code or Subtitle Edit → preview in mpv → repeat.

## Stage 5: Burn subtitles into final MP4

Use ffmpeg@7 with the full path. ffmpeg 8.x will fail with "No such filter: 'subtitles'".

### Minimal version (default styling)

```bash
/opt/homebrew/opt/ffmpeg@7/bin/ffmpeg -i input.mp4 \
  -vf "subtitles=filename=subtitle.srt" \
  -c:a copy output.mp4
```

**Note:** The `filename=` prefix is required in ffmpeg 8.x+ syntax. Older `subtitles=subtitle.srt` shorthand fails with "No option name near...".

### Styled version (recommended for product demos)

```bash
/opt/homebrew/opt/ffmpeg@7/bin/ffmpeg -i input.mp4 \
  -vf "subtitles=filename=subtitle.srt:force_style='FontName=PingFangSC\,FontSize=18\,PrimaryColour=&HFFFFFF&\,OutlineColour=&H000000&\,BorderStyle=1\,Outline=2\,MarginV=40'" \
  -c:a copy output.mp4
```

**Style parameter notes:**
- Commas inside `force_style` MUST be escaped as `\,` — ffmpeg uses unescaped commas as filter separators
- Font name: use `PingFangSC` (no space) on macOS for Chinese; spaces in font names cause parsing failures
- `FontSize=18` works well for 720p screen recordings; bump to 22-24 for 1080p
- `MarginV=40` is distance from bottom edge in pixels; increase to 60-80 if the recording shows a dock or important UI at the bottom
- `BorderStyle=1, Outline=2` produces white text with black outline (universally readable on any background)
- Colors are in `&HBBGGRR&` format (note: BGR, not RGB)

### Convenient alias

```bash
echo 'alias ffmpeg7="/opt/homebrew/opt/ffmpeg@7/bin/ffmpeg"' >> ~/.zshrc
source ~/.zshrc
```

Then `ffmpeg7 -i input.mp4 ...` instead of the full path.

## One-shot pipeline function

Drop this into `~/.zshrc` for `extract_subtitle video.mp4 → video.srt`:

```bash
extract_subtitle() {
  local video="$1"
  local prompt="${2:-Technical product demo}"
  local base="${video%.*}"

  ffmpeg -i "$video" -ar 16000 -ac 1 -c:a pcm_s16le "${base}.wav" -y || return 1
  whisper-cli -m ~/whisper-models/ggml-large-v3.bin \
    -l en -osrt -t 8 --prompt "$prompt" "${base}.wav" || return 1
  mv "${base}.wav.srt" "${base}.srt"
  rm "${base}.wav"
  echo "Subtitle generated: ${base}.srt"
}
```

Usage: `extract_subtitle demo.mp4 "Backed.AI demo, terms: PostgreSQL, MCP, BaaS"`

## Troubleshooting

**`Unknown encoder 'pcm_s161e'`** — Typo: should be `pcm_s16le` (L+E, not 1+E).

**`No such filter: 'subtitles'`** — You're using ffmpeg 8.x without libass. Install `ffmpeg@7` and use its full path: `/opt/homebrew/opt/ffmpeg@7/bin/ffmpeg`.

**`No option name near 'subtitle.srt'`** — ffmpeg 8.x+ requires explicit `filename=` prefix: `subtitles=filename=subtitle.srt`.

**`No option name near 'force_style=...'`** — Commas inside `force_style` are not escaped. Replace `,` with `\,` for every separator inside the style string.

**`command not found: whisper-cli`** — `whisper-cpp` not installed: `brew install whisper-cpp`.

**`command not found: whisper`** — `whisper` is the Python OpenAI package, NOT `whisper-cpp`. They're different tools. Install via `pipx install openai-whisper`, or use `whisper-cli` (the whisper.cpp CLI).

**mpv broken after uninstalling ffmpeg** — mpv depends on the main `ffmpeg` formula. Reinstall it (`brew install ffmpeg`); it can coexist with `ffmpeg@7` since the latter is keg-only.

**Whisper misspells product names / acronyms** — Add them to `--prompt`. The model uses prompt text as a strong vocabulary hint. Keep prompts under 200 tokens; longer prompts get truncated.

**Subtitles drift over long video** — Whisper.cpp occasionally has timestamp drift on >30 minute audio. Use `whisper-cli ... --max-len 80` to force shorter segments and re-anchor more frequently, or split the audio into chunks before transcription.

## When to use Screen Studio's built-in captions instead

If all of these are true, skip this skill and use Screen Studio directly:
- The `.screenstudio` project file is still available
- You don't need to edit caption text precisely (Screen Studio's caption editor is limited)
- The default animated caption styling fits the deliverable

This skill is the right choice when: the project file is gone, you need precise text control, you need consistent styling matching brand guidelines, or you want subtitles as a separate `.srt` file (not just burned-in).
