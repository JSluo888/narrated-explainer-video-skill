# Narrated Explainer Video · a Claude Skill

**One prompt in, a narrated, fully animated explainer MP4 out.**

[![Demo: Pathology foundation models in 27 seconds](assets/demo.gif)](assets/demo-pathology-foundation-models.mp4)

> The 27-second demo above was made end to end by Claude using only this skill: script, voiceover, subtitles, every animation, and the final render. No video editor, no paid APIs.
>
> ▶ **[Watch it with sound (MP4)](assets/demo-pathology-foundation-models.mp4)**

## What you get

- **1920×1080, 30 fps MP4** with burned-in subtitles, chapter labels, and a progress bar
- **A natural neural voice** ([Kokoro](https://huggingface.co/hexgrad/Kokoro-82M), `af_heart` by default), generated sentence by sentence so every visual lands on the word it illustrates
- **Custom motion graphics**, not templates: scanning slides, magnifier loupes, tiles flying into mosaics, count-ups, data pulses along paths, scatter plots that self-organize into clusters, gauges, survival curves drawing themselves
- **Deterministic, frame-by-frame rendering** in headless Chromium, encoded with ffmpeg, then checked with contact sheets and loudness QA before delivery

## Install

**Claude.ai / Claude desktop app**

1. Download [`narrated-explainer-video.zip`](narrated-explainer-video.zip).
2. Go to **Customize → Skills** ([claude.ai/customize/skills](https://claude.ai/customize/skills)), click **+**, choose **Create skill → Upload a skill**, and upload the zip.
3. Make sure code execution is on. The skill downloads the Kokoro voice model (~350 MB) from GitHub and a few Python packages from PyPI on first use, so your environment needs network access to those.

**Claude Code**

```bash
git clone https://github.com/JSluo888/narrated-explainer-video-skill.git
mkdir -p ~/.claude/skills
cp -r narrated-explainer-video-skill/narrated-explainer-video ~/.claude/skills/
```

## Use it

Just ask for a video:

- *"Make a 2-minute narrated explainer on how CRISPR base editing works."*
- *"Turn this abstract into a 45-second video I can post on LinkedIn."*
- *"Explain how a transformer processes a sentence in under 90 seconds, British male voice."*

Claude researches, writes the script, voices it, animates it, previews it, renders it, and hands you the MP4.

## Make it yours

A skill is just a Markdown file, and Claude is very good at editing it. Tell Claude what you want changed and ask it to update the skill:

- *"Update my narrated-explainer-video skill to use my brand colors and end with my logo."*
- *"Add a vertical 1080×1920 mode for Reels and Shorts."*
- *"Add soft background music under the narration."*
- *"Switch the default voice to `bm_george`."*

Pull requests with new motion vocabulary, voices, or formats are very welcome.

## How it works

```
script.py ──► Kokoro TTS (per sentence) ──► timeline.json
                                               │
anim.html  (SVG, every frame a pure function of time) ◄┘
   │
   ├─► preview contact sheets ──► fix layout issues
   └─► Playwright frame capture ──► ffmpeg encode + mux ──► QA ──► MP4
```

Because each frame is a pure function of time, the animation is perfectly synced to the narration and the render is reproducible.

## Credits

- Voice: [Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M) via [kokoro-onnx](https://github.com/thewh1teagle/kokoro-onnx)
- Rendering: [Playwright](https://playwright.dev) + [FFmpeg](https://ffmpeg.org)
- Demo facts: UNI, Chen et al., *Nature Medicine* 2024. The prediction values shown in the demo are illustrative.

Built by [Tianhao Luo](https://github.com/JSluo888). MIT License.
