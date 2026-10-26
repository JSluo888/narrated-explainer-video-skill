---
name: "narrated-explainer-video"
description: "Make an animated, narrated explainer video (MP4) using the Kokoro af_heart voice, synced subtitles, and SVG motion graphics rendered frame by frame."
---

# Narrated explainer video

Use this whenever the user asks for a video, animated walkthrough, or explainer with narration or voiceover. Output is a 1920x1080, 30 fps MP4 with a natural neural voice, burned-in subtitles, chapter labels and a progress bar.

**Default voice: Kokoro `af_heart` at speed 1.0.** Use it unless the user asks for a different voice (options are listed in section 2).

Pipeline: script -> voice (per sentence) -> timeline.json -> deterministic HTML/SVG animation driven by the timeline -> preview contact sheets -> frame capture with headless Chromium -> ffmpeg encode + mux -> QA -> deliver.

## 0. Setup (once per session)

```bash
pip install --break-system-packages kokoro-onnx soundfile
mkdir -p ~/tts && cd ~/tts
for f in kokoro-v1.0.onnx voices-v1.0.bin; do
  curl -sSL -o $f "https://github.com/thewh1teagle/kokoro-onnx/releases/download/model-files-v1.0/$f"
done
which ffmpeg
python3 -c "from playwright.sync_api import sync_playwright" || (pip install --break-system-packages playwright && playwright install chromium)
fc-list | grep -i inter   # Inter font is usually installed; otherwise fall back to a sans-serif
```

The model files come from GitHub releases (~350 MB). If the download is blocked, tell the user plainly; do not swap in a robotic voice silently.

## 1. Research and script

- Verify any current-state facts (latest products, figures, dates) with web search before writing; keep claims modest and dated.
- Aim for ~150 spoken words per minute. 5 to 6 minutes is plenty for a broad tour; shorter if the user asks.
- Write `script.py` as a list of scenes; each scene has an id, a chapter label and sentences. Each sentence has `sub` (subtitle text) and an optional `say` (what the voice reads) when pronunciation needs help.
- Keep on-screen text and subtitles light on em dashes; commas and periods read and sound more natural.

```python
SCENES = [
  ("intro", "Intro", [
    {"sub": "Every time you open a laptop, you're using a stack of hardware."},
  ]),
  ("memory", "03 · Memory & Storage", [
    {"sub": "Next to the CPU sits RAM, usually DDR5 today.",
     "say": "Next to the CPU sits RAM, usually D D R five today."},
  ]),
]
```

### Pronunciation rules (Kokoro + espeak)

Check tricky words before generating: `k.tokenizer.phonemize("word", "en-us")`.

- Common all-caps acronyms (CPU, GPU, AI, RAM, SSD) are read fine.
- Spell out unusual ones with spaces: `D D R five`, `N V M E`, `P C I Express`, `H B M four`, `U P S`, `P U E`, `N P U`.
- Plurals of spelled acronyms: `T P U's` (not `T P Us`, which reads as "us").
- Brand fixes: `En-vidia` for NVIDIA, `En-vee link` for NVLink.
- Write numbers and years as words in `say`: `twenty twenty six`, `one point six terabit`, `two hundred and eight billion`.
- Replace colons/dashes in `say` with periods or commas so the voice pauses naturally.

## 2. Voice (tts.py)

Generate each sentence separately so every sentence has exact start/end times for animation cues and subtitles.

```python
import json, numpy as np, soundfile as sf
from kokoro_onnx import Kokoro
from script import SCENES
import os
TTS = os.path.expanduser('~/tts')   # where the setup step downloaded the model files
k = Kokoro(f'{TTS}/kokoro-v1.0.onnx', f'{TTS}/voices-v1.0.bin')
SR=24000; VOICE='af_heart'; SPEED=1.0
LEAD=0.8; GAP=0.32; SCENE_GAP=0.9; TAIL=1.6
out=[np.zeros(int(LEAD*SR),dtype=np.float32)]; t=LEAD; timeline=[]
for si,(sid,chap,sents) in enumerate(SCENES):
    sc={"id":sid,"chapter":chap,"start":t,"sentences":[]}
    for j,s in enumerate(sents):
        a,sr=k.create(s.get("say",s["sub"]),voice=VOICE,speed=SPEED,lang="en-us")
        a=a.astype(np.float32)
        idx=np.where(np.abs(a)>0.01)[0]
        a=a[max(0,idx[0]-240):min(len(a),idx[-1]+1200)]   # trim silence
        d=len(a)/SR
        sc["sentences"].append({"start":t,"end":t+d,"sub":s["sub"]})
        out.append(a); t+=d
        g=GAP if j<len(sents)-1 else (SCENE_GAP if si<len(SCENES)-1 else TAIL)
        out.append(np.zeros(int(g*SR),dtype=np.float32)); t+=g
    sc["end"]=t; timeline.append(sc)
audio=np.concatenate(out); audio=audio/max(1e-6,np.abs(audio).max())*0.89
sf.write('narration.wav',audio,SR)
json.dump({"duration":t,"scenes":timeline},open('timeline.json','w'),indent=1)
print("TOTAL",round(t,1))
```

Run it in the background (`nohup python3 tts.py > tts.log 2>&1 &`); a 6-minute script takes ~2 minutes on CPU. Then export the timeline for the page: `window.TL = <timeline.json>` into `timeline.js`.

Other voices if the user asks: `af_bella`, `af_nicole` (US female), `am_michael`, `am_fenrir`, `am_puck` (US male), `bf_emma`, `bf_isabella` (UK female), `bm_george`, `bm_lewis` (UK male).

## 3. Animation page (deterministic)

One `anim.html` with an `<svg viewBox="0 0 1920 1080">`, static `<defs>` (background gradient, faint grid pattern, glow filter, any gradients/clipPaths), a `<g id="content">`, a `<g id="hud">`, plus HTML overlays for the chapter pill (top-left), a small brand label (top-right) and subtitles (bottom). Scripts: `timeline.js`, `core.js`, scene files.

Rules:
- Everything is a pure function of time. `renderAt(T)` rebuilds `content.innerHTML` from scratch each frame. No CSS animations, no `Date.now()`, no `Math.random()`; use a seeded `rnd(i)` and `Math.floor(t*rate)` for blinking.
- Each scene is `FN[id] = (t, S, E, D) => svgString`, where `t` is local time, `S[j]`/`E[j]` are local start/end of sentence j, `D` is scene length. Cue visuals to sentences (`ep(t, S[1]+0.3)`), and to words inside long sentences by fraction (`S[2] + (E[2]-S[2])*0.6`).
- Safe zone for visuals: y 120 to 860. Subtitles own the bottom ~200 px; chapter pill owns the top-left.
- Look: dark navy background (#0A0F1E with a radial lift), Inter font, one accent color per concept (e.g. CPU cyan #38BDF8, memory violet #A78BFA, GPU green #34D399, power amber #FBBF24, heat red #F87171, cooling blue #60A5FA, network pink #F472B6, storage teal #2DD4BF). Big readable labels (26 to 56 px; 100+ px for hero numbers).
- Motion vocabulary: staggered fade/slide-ins, callouts with leader lines, flowing pulses along paths (data, power, coolant, air), count-ups, scenes that split into panels which crossfade per sentence (`win(t, a, b)`), rotating fans, blinking LEDs.
- Scene fade in 0.5 s at start, fade out 0.42 s at end; final 1.2 s fades to black.

### core.js essentials

```js
const clamp=(x,a=0,b=1)=>x<a?a:x>b?b:x, lerp=(a,b,u)=>a+(b-a)*u;
const eo=u=>{u=clamp(u);return 1-Math.pow(1-u,3)};
const eio=u=>{u=clamp(u);return u<.5?4*u*u*u:1-Math.pow(-2*u+2,3)/2};
const pr=(t,a,d=0.6)=>clamp((t-a)/d), ep=(t,a,d=0.6)=>eo((t-a)/d);
const win=(t,a,b,fi=0.5,fo=0.45)=>Math.min(ep(t,a,fi),1-pr(t,b,fo));
const rnd=i=>{const x=Math.sin(i*127.1+311.7)*43758.5453;return x-Math.floor(x)};
const f=n=>Math.round(n*100)/100, pp=p=>f(p[0])+' '+f(p[1]);
function hexA(h,a){const n=parseInt(h.slice(1),16);return `rgba(${n>>16&255},${n>>8&255},${n&255},${a})`;}
function mix(h1,h2,u){const a=parseInt(h1.slice(1),16),b=parseInt(h2.slice(1),16);const c=s=>Math.round(lerp(a>>s&255,b>>s&255,clamp(u)));return `rgb(${c(16)},${c(8)},${c(0)})`;}
// rect/circ/line/path/text/grp(inner,{x,y,s,cx,cy,op,clip}) return SVG strings and return '' when op<=0.002.
// polyPt(pts,u): point at fraction u along a polyline.
function pulses(pts,t,o={}){const n=o.n||6,sp=o.speed||0.5;let s='';for(let k=0;k<n;k++){const u=(((t*sp+k/n+(o.phase||0))%1)+1)%1;const [x,y]=polyPt(pts,u);const fade=Math.min(1,u*7,(1-u)*7);const col=typeof o.col==='function'?o.col(u):(o.col||'#fff');s+=circ(x,y,o.r||6,{fill:col,op:(o.op??1)*fade,filter:o.filter});}return s;}
// callout(x,y,lx,ly,label,col,p,{sub,anchor,op}): dot + growing leader line + label that fades in as p goes 0->1.

// Subtitles: split each sentence into chunks of <=74 chars, prefer breaking after , : ; . (left part >= 26 chars),
// never end a chunk on a weak word (the, a, of, to, into, and, in, on, with, for, PCI...), merge tails < 16 chars.
// Time each chunk by its share of the sentence's characters.

function renderAt(T){
  const SC=TL.scenes; let si=SC.findIndex(s=>T>=s.start&&T<s.end); if(si<0) si=T<SC[0].start?0:SC.length-1;
  const sc=SC[si], t=T-sc.start, D=sc.end-sc.start;
  const S=sc.sentences.map(x=>x.start-sc.start), E=sc.sentences.map(x=>x.end-sc.start);
  const op=Math.min(si===0?ep(T,0,0.6):ep(t,0,0.5), si===SC.length-1?1:1-pr(t,D-0.42,0.42));
  content.innerHTML=`<g opacity="${f(op)}">${FN[sc.id](t,S,E,D)}</g>`;
  // chapter pill (hidden on intro), progress bar at y=1074 with chapter ticks, end fade, current subtitle chunk
}
```

Subtitle CSS: centered at bottom 44px, 38px Inter 550, line-height 1.3, max-width 1680px, background rgba(3,6,14,.82), padding 12px 32px, radius 16px. Show the latest chunk whose start <= T; fade in over 0.18 s; hide at scene change.

Add `renderAt(parseFloat(new URLSearchParams(location.search).get('t')||'0'))` at the end so a single frame can be opened directly.

## 4. Preview before rendering

Render 6 frames per contact sheet (one per sentence or two per scene) and look at every scene. Fix overlaps, clipped labels, text colliding with subtitles, panels that appear too late for the narration, and long subtitle chunks before the full render.

```python
# preview.py out.png t1,t2,...
import sys
from playwright.sync_api import sync_playwright
from PIL import Image
ts=[float(x) for x in sys.argv[2].split(',')]; errs=[]; ims=[]
with sync_playwright() as p:
    b=p.chromium.launch(); pg=b.new_page(viewport={'width':1920,'height':1080})
    pg.on('pageerror',lambda e:errs.append(str(e)))
    pg.goto('file://'+__import__('os').path.abspath('anim.html')); pg.wait_for_timeout(300)
    for t in ts:
        pg.evaluate(f'renderAt({t})'); pg.screenshot(path='/tmp/_f.png')
        ims.append(Image.open('/tmp/_f.png').convert('RGB').resize((960,540)))
    b.close()
sheet=Image.new('RGB',(1920,540*((len(ims)+1)//2)))
for i,im in enumerate(ims): sheet.paste(im,((i%2)*960,(i//2)*540))
sheet.save(sys.argv[1]); print('errors:',errs)
```

Also print all subtitle chunks once (`CUES.map(c=>c.text)`) and check none split a phrase awkwardly.

## 5. Full render (two workers)

CDP JPEG capture runs ~10 frames/s per worker at 1080p, so a 6-minute video takes ~9 minutes with 2 workers.

```python
# render.py k K
import sys, json, math, base64, subprocess, os
from playwright.sync_api import sync_playwright
k=int(sys.argv[1]); K=int(sys.argv[2]); FPS=30
N=math.ceil(json.load(open('timeline.json'))['duration']*FPS); a=N*k//K; b=N*(k+1)//K
ff=subprocess.Popen(['ffmpeg','-y','-loglevel','error','-f','image2pipe','-framerate',str(FPS),'-c:v','mjpeg','-i','-',
  '-c:v','libx264','-preset','veryfast','-crf','19','-pix_fmt','yuv420p','-r',str(FPS),f'seg{k}.mp4'],stdin=subprocess.PIPE)
with sync_playwright() as p:
    br=p.chromium.launch(); pg=br.new_page(viewport={'width':1920,'height':1080})
    pg.goto('file://'+os.path.abspath('anim.html')); pg.wait_for_timeout(500)
    cdp=pg.context.new_cdp_session(pg)
    for i in range(a,b):
        pg.evaluate(f'renderAt({i/FPS})')
        ff.stdin.write(base64.b64decode(cdp.send('Page.captureScreenshot',{'format':'jpeg','quality':93,'optimizeForSpeed':True})['data']))
        if (i-a)%300==0: print(k,i-a,b-a,flush=True)
    br.close()
ff.stdin.close(); ff.wait(); print('done',k,flush=True)
```

Launch `python3 render.py 0 2 > r0.log 2>&1 &` and `python3 render.py 1 2 > r1.log 2>&1 &`, then wait with a background `until grep -q '^done' r0.log && grep -q '^done' r1.log; do sleep 10; done` (also break on `Traceback`). Then:

```bash
printf "file 'seg0.mp4'\nfile 'seg1.mp4'\n" > list.txt
ffmpeg -y -loglevel error -f concat -safe 0 -i list.txt -i narration.wav -map 0:v -map 1:a \
  -c:v copy -c:a aac -b:a 192k -shortest -movflags +faststart Final_Title.mp4
```

## 6. QA and delivery

- `ffprobe` the result (1920x1080, 30 fps, h264 + aac, expected duration).
- Pull ~8 frames from the final MP4 (including the segment boundary at N/2/30 s) into a contact sheet and look at them.
- `ffmpeg -i Final.mp4 -af volumedetect -vn -f null -` (expect max around -1 dB).
- Send with SendUserFile (`display: render`). Reply briefly: length, what it covers, sources for any current facts, and one offer (shorter cut, vertical 1080x1920 version for phones, or a different voice).