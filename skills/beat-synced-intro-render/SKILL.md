---
name: procedural-beat-synced-brand-intro
description: Produce a deterministic 2–2.5 min, 1920x1080@30fps, beat-synced minimalist motion-graphics brand intro entirely procedurally on a CPU-only GitHub Actions runner (4 vCPU / 16 GB, no GPU, Python 3.11, ffmpeg). Covers reference-video shot/color/motion analysis, MP3 beat + section analysis, frame-accurate kinetic-typography rendering piped to ffmpeg without PNG round-trips, and output verification. Output is one H.264 MP4 plus an analysis markdown and contact-sheet JPGs.
---

# Procedural Beat-Synced Brand Intro

## When to use

Use when all of the following hold:

- A brand intro (2–2.5 min) must be generated from code on a headless Linux CI runner with no GPU and no AI video model.
- A source MP3 drives timing (BPM, onsets, sections) and a reference MP4 must be analyzed for cut rhythm, palette, and motion vocabulary.
- The visual language is minimalist: near-black background, off-white type, one acid-lime accent, kinetic typography + primitive geometry, cubic easing only.
- The output must be byte-stable given the same inputs, and must be verifiable by machine.

Do **not** use when: a GPU is available (use a real compositor), when the deliverable is interactive, or when audio is not the timing authority.

---

## Rules

Hard constraints. Every rule is objectively checkable.

**The eight killers — these break this render class more than anything else.**

1. **P1 — Never give ffmpeg a piped stderr.** `stderr=subprocess.PIPE` on a long render deadlocks at ~64 KB. Redirect `stderr` to an open *file* handle, and pass `-nostdin -loglevel error`. Checkable: no `PIPE` for `stderr` anywhere in the renderer; a render of ≥3000 frames completes.
2. **P2 — All timing lives on integer frame indices.** No `if t >= cut_time` on floats, ever. Times are converted to frames exactly once, at timeline-compile time, via `int(round(t * FPS))`. Checkable: `timeline.json` contains only integer frame fields; no `float` comparison against time in the renderer.
3. **P3 — Encode 4:2:0 limited-range deliberately.** Chroma subsampling deletes 1-px lime strokes and shifts near-black. Minimum stroke width is 3 px at 1920x1080; `#0A0A0A`–`#141414` is the allowed background band; set `-pix_fmt yuv420p -color_range tv -colorspace bt709 -color_primaries bt709 -color_trc bt709` on encode. Checkable: no geometry thinner than 3 px in the scene graph.
4. **P4 — Pin every source of nondeterminism.** Pinned package versions, pinned x264 `threads=N`, `-fflags +bitexact -flags:v +bitexact -flags:a +bitexact -map_metadata -1`, `SOURCE_DATE_EPOCH`, `TZ=UTC`, `PYTHONHASHSEED=0`, explicit RNG seeds, no `datetime.now()`, no set/dict iteration feeding draw order. Checkable: two consecutive runs of the same commit produce identical `sha256` for the MP4.
5. **P5 — Audio analysis and audio mux must consume byte-identical PCM.** Decode the MP3 **once** to `build/master.wav` (48 kHz, stereo, pcm_s24le), then derive the mono analysis WAV from that same file. This neutralizes MP3 decoder delay. Checkable: `sha256(build/master.wav)` is recorded in `analysis/audio.json` and reused.
6. **P6 — Never buffer more than one frame.** 4500 frames × 6.22 MB = 28 GB. Allocate one `np.empty((1080,1920,3), np.uint8)` canvas and one supersample scratch buffer, reuse both for the whole render. Checkable: peak RSS < 2 GB during render.
7. **P7 — Resolve fonts by absolute path, never by family name.** `ImageFont.truetype("Inter")` resolves via fontconfig and silently returns different files on different images. Use a vendored TTF path + SHA-256 assertion, with an absolute DejaVu path as the declared fallback, and lay out from measured metrics so the fallback still composes. Checkable: `font.path` is asserted to equal the pinned absolute path at startup.
8. **P8 — Cache all rasterized text; never re-rasterize per frame.** Re-rendering 1080p text every frame blows the 3-hour budget. Pre-rasterize each text run once into an RGBA numpy tile (2× supersampled, downsampled with `INTER_AREA`), then per-frame only translate + alpha-composite. Checkable: render stage wall time < 20 min for 4500 frames.

**Timeline and sync**

9. `timeline.json` is the single source of truth. No stage after Step 5 recomputes a time, beat, or section boundary.
10. Render time is `t = frame_index / 30.0`. `frame_index` is the loop variable. Audio PTS and video PTS both start at 0; the mux step must not resample or shift.
11. Video duration is exactly `duration_frames / 30.0` seconds, enforced with `-t <exact decimal>` on both the render and the mux.
12. Duration is chosen from the music: the last detected section boundary ≤ 150.0 s, clamped to ≥ 120.0 s, snapped to a beat.
13. `cut_frames` are deduplicated and enforce a minimum gap of 6 frames (200 ms). Adjacent onsets closer than that collapse to the earlier one.
14. Every visual event (text in/out, shape entrance, accent flash) references a frame index drawn from `cut_frames` or `beat_frames`, never from a free-floating constant.
15. Any animation that must complete before a cut starts at `cut_frame - lead_frames` and is asserted to finish at or before `cut_frame`.
16. Cuts are hard switches (segment index), not crossfades, unless the reference analysis shows the source uses dissolves.

**Pipeline mechanics**

17. Frames reach ffmpeg as `rawvideo`/`rgb24` on stdin. No PNG, no JPEG, no intermediate frame files anywhere.
18. Canvas dtype is `uint8`, C-contiguous, shape `(1080, 1920, 3)`. `np.ascontiguousarray` before `.tobytes()`.
19. `Popen` uses `bufsize >= 4 * FRAME_BYTES` so partial writes are handled by the buffered writer, and `stdin.close()` is always followed by `wait()` with a return-code check.
20. A nonzero ffmpeg return code raises with the last 4 KB of the stderr log file attached. Silent failure is forbidden.
21. Encoding is two-pass: render video-only to `render/video.mp4`, then mux with `-c:v copy`. A render failure must not corrupt or partially write the deliverable.
22. Alpha compositing uses integer math: `out = (bg * (255 - a) + fg * a + 127) // 255` on `uint16`. No float blending in the hot loop.
23. Geometric AA is analytic: signed-distance fields with a 1-px smoothstep, or supersampled local bounding boxes. No full-frame 2× supersampling of the whole canvas.
24. Easing functions are exactly three: `ease_out_cubic(u)=1-(1-u)**3`, `ease_in_cubic(u)=u**3`, `ease_in_out_cubic(u)=4u³ if u<0.5 else 1-(-2u+2)³/2`. `u` is clamped to `[0,1]`. No springs, no bounces, no overshoot.

**Analysis**

25. Reference shot detection uses PySceneDetect `ContentDetector` with `--threshold 27 --min-scene-len 15`. If it yields > 120 shots, raise the threshold until ≤ 120 — music videos over-segment on strobe cuts.
26. Per-shot color is the mean of the shot's middle frame resized with `cv2.resize(..., INTER_AREA)`, reported in Lab and sRGB hex.
27. Per-shot motion is the mean absolute difference of consecutive 160x90 grayscale frames, plus `cv2.phaseCorrelate` for dominant pan direction.
28. BPM is reported as a candidate set (0.5×, 1×, 2×) with the chosen value selected by maximum autocorrelation of the onset envelope at the beat lag; the beat-interval standard deviation is recorded as a confidence measure.
29. Section segmentation uses a 2048-hop RMS + log-mel(8) + spectral-centroid feature stack, novelty = sum of positive first differences, peaks snapped to the nearest beat.
30. Audio analysis is deterministic: `OPENBLAS_NUM_THREADS=1`, `OMP_NUM_THREADS=1`, `PYTHONHASHSEED=0` for the analysis job.

**Verification**

31. Verification never uses `blackdetect` — the background is intentionally near-black. Use sampled-frame luminance statistics and inter-frame difference instead.
32. The muxed output's audio is re-analyzed with the same onset detector; every planned onset frame must appear within ±1 frame.
33. The deliverable's `ffprobe` `start_time` is `0.000000` for both the video and audio streams.

---

## Steps

### 1. Scaffold and pin the environment

```
brand-intro/
  .github/workflows/render.yml
  pyproject.toml | requirements.lock.txt
  assets/fonts/            # vendored TTFs + FONTS.lock
  src/{audio_analysis,reference_analysis,timeline,render,verify}.py
  src/scene/{layers,easing,text,shapes,composite}.py
  build/                   # scratch: master.wav, analysis.wav  (gitignored)
  render/                  # video.mp4 (gitignored)
  out/                     # brand_intro.mp4, analysis.md, sheets/*.jpg
  analysis/
```

`requirements.lock.txt` — known-good CPU-only set:

```
numpy==1.26.4
scipy==1.13.1
librosa==0.10.2.post1
numba==0.60.0
llvmlite==0.43.0
soundfile==0.12.1
Pillow==10.4.0
opencv-python-headless==4.10.0.84
scenedetect==0.6.5
jsonschema==4.23.0
```

Runner: `ubuntu-24.04` (ffmpeg 6.x, so `-fps_mode` is available), `timeout-minutes: 170`, cache `~/.cache/pip`.

Workflow env: `TZ=UTC`, `PYTHONHASHSEED=0`, `SOURCE_DATE_EPOCH=1700000000`, `LC_ALL=C.UTF-8`.

**Verify:** `ffmpeg -version | head -1` and `python -c "import librosa, cv2, PIL; print(librosa.__version__)"` both succeed; `pip check` is clean.

### 2. Decode the MP3 exactly once

```bash
mkdir -p build
ffmpeg -hide_banner -nostdin -y -loglevel error -i "ptasinski RJ Pasin - life force.mp3" \
  -vn -ac 2 -ar 48000 -c:a pcm_s24le build/master.wav
ffmpeg -hide_banner -nostdin -y -loglevel error -i build/master.wav \
  -vn -ac 1 -ar 48000 -c:a pcm_f32le build/analysis.wav
sha256sum build/master.wav
```

Every later stage reads `build/master.wav` or `build/analysis.wav`. Never re-decode the MP3.

**Verify:** `ffprobe -v error -show_entries stream=sample_rate,channels -of csv build/master.wav` prints `48000,2`; the recorded SHA-256 matches on re-run.

### 3. Audio analysis → `analysis/audio.json`

```python
import librosa, numpy as np
HOP = 512
y, sr = librosa.load("build/analysis.wav", sr=None, mono=True)   # sr == 48000

env = librosa.onset.onset_strength(y=y, sr=sr, hop_length=HOP, aggregate=np.median)
tempo = float(librosa.feature.rhythm.tempo(
    onset_envelope=env, sr=sr, hop_length=HOP, start_bpm=120.0, std_bpm=1.0)[0])

_, beat_f = librosa.beat.beat_track(onset_envelope=env, sr=sr, hop_length=HOP,
                                    bpm=tempo, trim=False)
onset_f = librosa.onset.onset_detect(
    onset_envelope=env, sr=sr, hop_length=HOP, units='frames',
    pre_max=3, post_max=3, pre_avg=3, post_avg=5, delta=0.06, wait=4, backtrack=False)

t_beat  = librosa.frames_to_time(beat_f,  sr=sr, hop_length=HOP)
t_onset = librosa.frames_to_time(onset_f, sr=sr, hop_length=HOP)

# bands
S = np.abs(librosa.stft(y, n_fft=2048, hop_length=HOP))
freqs = librosa.fft_frequencies(sr=sr, n_fft=2048)
BANDS = {"sub": (20,120), "low": (120,500), "mid": (500,2000),
         "high": (2000,8000), "air": (8000,16000)}
band_energy = {k: S[(freqs>=lo)&(freqs<hi)].mean(axis=0).tolist()
               for k,(lo,hi) in BANDS.items()}
```

Sections: hop 2048, stack `[rms, spectral_centroid, log-mel(8)]` z-scored, novelty = `np.maximum(np.diff(feat, axis=1), 0).sum(axis=0)`, smooth with a 9-frame Hann, peak-pick with `scipy.signal.find_peaks(height=p75, distance=16)`, snap each boundary to the nearest beat.

Confidence: `np.std(np.diff(t_beat))` — record it; if > 0.03 s, flag the BPM as low-confidence and fall back to `librosa.feature.rhythm.tempo(..., start_bpm=tempo)`.

**Verify:** `analysis/audio.json` validates against its JSON Schema; `abs(tempo - 60/np.median(np.diff(t_beat))) < 1.0`; section boundaries are strictly increasing and within `[0, duration]`.

### 4. Reference-video analysis → `analysis/reference.md` + contact sheets

```bash
scenedetect -i ref.mp4 detect-content --threshold 27 --min-scene-len 15 \
  list-scenes -f analysis/shots.csv -o analysis/
```

```bash
# bulk grid sheet, one row per 40 s
ffmpeg -hide_banner -nostdin -y -loglevel error -i ref.mp4 \
  -vf "fps=1/4,scale=384:-1,tile=5x4:padding=6:margin=6:color=0x0A0A0A" \
  -frames:v 1 -q:v 3 analysis/sheets/grid_%02d.jpg
```

Labeled per-shot sheets with Pillow: for each shot, extract the middle frame with `-ss <mid> -frames:v 1`, draw `#NNN  t=MM:SS.mmm` in off-white at the tile's bottom-left, assemble into a 4×3 grid, save at `-q:v 3` (`quality=85`, `optimize=True`).

Per-shot metrics in Python:

```python
mid = frame_at(shot.mid)                       # BGR uint8
small = cv2.resize(mid, (32, 18), interpolation=cv2.INTER_AREA)
lab = cv2.cvtColor(small, cv2.COLOR_BGR2LAB).reshape(-1,3).mean(0)
# motion
gray = [cv2.cvtColor(cv2.resize(f,(160,90),interpolation=cv2.INTER_AREA),
                     cv2.COLOR_BGR2GRAY).astype(np.int16) for f in shot_frames]
mad  = float(np.mean([np.abs(gray[i+1]-gray[i]).mean() for i in range(len(gray)-1)]))
shift, _ = cv2.phaseCorrelate(gray[0].astype(np.float32), gray[-1].astype(np.float32))
```

`analysis/reference.md` sections: shot count, median shot length, cut-rate histogram, dominant palette (k-means k=5 in Lab over shot means, reported as hex), motion distribution, dissolve ratio, and a "vocabulary" paragraph naming the 3–5 recurring compositional devices.

**Verify:** `analysis/shots.csv` has ≥ 5 rows and ≤ 120 rows; every shot has a nonzero duration; each generated sheet file exists and `ffprobe` reports width > 1000.

### 5. Compile `timeline.json`

```python
FPS = 30
to_f = lambda t: int(round(t * FPS))

beats  = sorted({to_f(t) for t in t_beat})
onsets = sorted({to_f(t) for t in t_onset})

cuts, last = [], -10**9
for f in onsets:
    if f - last >= 6:
        cuts.append(f); last = f

end = max([to_f(s.end) for s in sections if s.end <= 150.0], default=120*FPS)
end = max(min(end, 150*FPS), 120*FPS)
end = min(beats, key=lambda b: abs(b - end))
duration_frames = end
```

Emit, then validate with `jsonschema`:

```json
{"version":1,"fps":30,"duration_frames":4140,"duration_s":"138.000000",
 "bpm":128.0,"bpm_confidence":0.004,"seed":1700000000,
 "beat_frames":[...],"onset_frames":[...],"cut_frames":[...],
 "sections":[{"start_frame":0,"end_frame":360,"label":"intro","rms":0.11,
              "band_energy":{"sub":0.02,"low":0.05,"mid":0.08,"high":0.03,"air":0.01}}],
 "master_wav_sha256":"...","fonts":{"display":"assets/fonts/...","sha256":"..."}}
```

**Verify:** schema validation passes; `all(a < b for a,b in zip(cut_frames, cut_frames[1:]))`; `all(f - p >= 6 for p,f in zip(cut_frames, cut_frames[1:]))`; `duration_frames % 1 == 0` and `duration_frames/30` is in `[120,150]`.

### 6. Acquire fonts

Preferred: vendor the TTF into `assets/fonts/` (OFL fonts are redistributable) and record it in `assets/fonts/FONTS.lock`:

```
SpaceGrotesk-Bold.ttf  <sha256>  https://github.com/google/fonts @ <commit-sha>
```

Startup assertion:

```python
DISPLAY = Path("assets/fonts/SpaceGrotesk-Bold.ttf")
FALLBACK = Path("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf")
path = DISPLAY if DISPLAY.exists() and sha256(DISPLAY) == LOCK else FALLBACK
assert path.is_absolute() and path.exists()
font = ImageFont.truetype(str(path), size, layout_engine=ImageFont.Layout.RAQM)
```

If the vendored file is absent (e.g. Git LFS not fetched), fetch at the pinned commit URL over HTTPS, verify SHA-256, write to `~/.cache/brand-intro-fonts/<sha>/`. On any failure, fall back to DejaVu and record the degradation in `analysis/reference.md`.

Layout must be metric-driven, never hardcoded:

```python
def fit_size(text, max_w, hi=420, lo=24):
    while lo < hi:
        mid = (lo + hi + 1)//2
        f = ImageFont.truetype(str(path), mid, layout_engine=ImageFont.Layout.RAQM)
        if f.getlength(text) <= max_w: lo = mid
        else: hi = mid - 1
    return lo
```

Letter-spacing: Pillow has no tracking. For all-caps display runs, draw per glyph at `x += font.getlength(ch) + tracking`; accept the loss of pair kerning and compensate with a small negative tracking on `AV`, `TA`, `LT` pairs. Cache the resulting RGBA tile.

**Verify:** `font.path == str(path)`; `fit_size` output produces `getlength(text) <= max_w` for every text run in the scene graph; the render log records which font was used.

### 7. Build the renderer

Scene graph: a list of layers, each `{kind, start_frame, end_frame, ease, from, to, ...}`, sorted by a stable key. Per frame, walk the layers whose window contains `frame_index`.

```python
FRAME_BYTES = 1920*1080*3
canvas = np.empty((1080, 1920, 3), np.uint8)   # reused for all frames

FFMPEG = [
 "ffmpeg","-hide_banner","-nostdin","-y","-loglevel","error",
 "-f","rawvideo","-pix_fmt","rgb24","-s","1920x1080","-r","30","-i","pipe:0",
 "-an","-c:v","libx264","-preset","slow","-crf","16",
 "-pix_fmt","yuv420p","-color_range","tv",
 "-colorspace","bt709","-color_primaries","bt709","-color_trc","bt709",
 "-x264-params","threads=4:scenecut=0:open-gop=0:keyint=60:min-keyint=60:bframes=2:ref=3:aq-mode=2",
 "-r","30","-fps_mode","cfr",
 "-fflags","+bitexact","-flags:v","+bitexact","-map_metadata","-1",
 "-t", f"{duration_frames/30:.6f}",
 "render/video.mp4",
]

with open("render/ffmpeg.log","wb") as errf:            # file, NOT a pipe  (P1)
    proc = subprocess.Popen(FFMPEG, stdin=subprocess.PIPE,
                            stdout=subprocess.DEVNULL, stderr=errf,
                            bufsize=FRAME_BYTES*4)
    try:
        for i in range(duration_frames):
            draw_frame(i, canvas)                       # writes into `canvas`
            proc.stdin.write(canvas.tobytes())
        proc.stdin.close()
    except BrokenPipeError:
        errf.flush()
        raise RuntimeError(Path("render/ffmpeg.log").read_text()[-4000:])
    if proc.wait() != 0:
        raise RuntimeError(Path("render/ffmpeg.log").read_text()[-4000:])
```

Layer implementations:

- **Text**: blit a cached RGBA tile. Position from easing. Tiles are built at 2× then `cv2.resize(tile, None, fx=0.5, fy=0.5, interpolation=cv2.INTER_AREA)` once.
- **Rect / line**: numpy slice assignment; enforce ≥ 3 px thickness (P3).
- **Circle / ring**: `yy, xx = np.ogrid[...]` over the layer's bounding box only; `d = np.sqrt((xx-cx)**2 + (yy-cy)**2)`; `alpha = np.clip((r + 0.5 - d), 0, 1)`.
- **Composite**: integer blend, bounding-box local only:

```python
def blend(dst, x, y, rgb, a):                 # a: uint8 HxW, 0..255
    sub = dst[y:y+a.shape[0], x:x+a.shape[1]]
    a3a3 = a[..., None].astype(np.uint16)
    sub[:] = ((sub.astype(np.uint16) * (255 - a3) +
               np.asarray(rgb, np.uint16) * a3 + 127) // 255).astype(np.uint8)

Easing and progress, clamped, no time floats:

    EASE = {"out":  lambda u: 1 - (1 - u)**3,
            "in":   lambda u: u**3,
            "inout": lambda u: 4*u**3 if u < 0.5 else 1 - (-2*u + 2)**3 / 2}

    def progress(layer, i):
        n = layer["end_frame"] - layer["start_frame"]
        u = 0.0 if n <= 0 else (i - layer["start_frame"]) / n
        return EASE[layer["ease"]](min(1.0, max(0.0, u)))

    def lerp(a, b, u):
        return a + (b - a) * u

Cut enforcement — the renderer reads the cut list, never the clock:

    import bisect
    def segment_at(i):
        return bisect.bisect_right(CUT_FRAMES, i) - 1

    for L in LAYERS:                                   # compile-time assertion
        seg = segment_at(L["start_frame"])
        seg_end = (CUT_FRAMES[seg+1] - 1) if seg + 1 < len(CUT_FRAMES) else DURATION_FRAMES - 1
        assert L.get("allow_straddle") or L["end_frame"] <= seg_end, L["id"]

This is the actual mechanism that guarantees cuts land on onsets. Cut times are converted to integer frame indices exactly once, in Step 5. Every layer's window is clamped to its segment's frame range and asserted at compile time. The segment switch at `frame_index == cut_frame` is a hard array reset — the canvas is re-filled with the background color and the layer list is re-filtered — not a blend. Since no layer is permitted to straddle a boundary and no interpolation reads across it, no rendered frame can sit between two planned frames. There is nothing left to round.

**Verify:** run the renderer with `--assert-only`; every layer passes the straddle assertion; `bisect` on `CUT_FRAMES` returns the expected segment for a sample of 20 frames either side of each cut.

### 8. Mux audio onto the rendered video

    ffmpeg -hide_banner -nostdin -y -loglevel error \
      -i render/video.mp4 -i build/master.wav \
      -map 0:v:0 -map 1:a:0 -c:v copy \
      -c:a aac -b:a 320k -ar 48000 -ac 2 \
      -fflags +bitexact -flags:v +bitexact -flags:a +bitexact \
      -movflags +faststart -map_metadata -1 \
      -t 138.000000 out/brand_intro.mp4

`-c:v copy` means the video bitstream is bit-identical to the render — no second generation of lossy encoding and no PTS rewriting. `-t` is `duration_frames/30` formatted `%.6f`, the same string the render step used, so the container duration is exactly the plan. Both inputs start at PTS 0, so `-copyts` is unnecessary and would be harmful.

Sync is guaranteed by construction, not by measurement: there is one `master.wav`, it is never resampled, the video is CFR at 30/1, and the mux does not touch the video stream. The only remaining sync risk is a wrong `duration_frames`, which Step 9 catches.

**Verify:** the mux exits 0; `ffprobe` reports `-c:v copy` produced an unchanged `nb_frames` versus `render/video.mp4`.

### 9. Verify the deliverable

    ffprobe -v error -count_frames -select_streams v:0 \
      -show_entries stream=codec_name,width,height,r_frame_rate,pix_fmt,start_time,nb_read_frames,duration \
      -of json out/brand_intro.mp4
    ffprobe -v error -select_streams a:0 \
      -show_entries stream=codec_name,sample_rate,channels,start_time,duration \
      -of json out/brand_intro.mp4
    ffmpeg -hide_banner -nostdin -y -loglevel error -i out/brand_intro.mp4 \
      -vn -ac 1 -ar 48000 -c:a pcm_f32le build/out_audio.wav

Structural checks, each written into `out/verify.json`:

    checks = {
      "v_codec":      v["codec_name"] == "h264",
      "resolution":   (v["width"], v["height"]) == (1920, 1080),
      "fps":          v["r_frame_rate"] == "30/1",
      "pix_fmt":      v["pix_fmt"] == "yuv420p",
      "nb_frames":    int(v["nb_read_frames"]) == DURATION_FRAMES,
      "v_start_zero": abs(float(v["start_time"])) < 1e-6,
      "a_start_zero": abs(float(a["start_time"])) < 1e-6,
      "audio_48k":    (a["sample_rate"], a["channels"]) == ("48000", 2),
      "duration":     abs(float(v["duration"]) - DURATION_FRAMES/30) < 1.0/30,
    }

Content checks that actually catch a broken render — a green build that ships a black video is the most common failure of this pipeline:

    # 1. sampled luminance band. NOT blackdetect: the background is intentionally near-black.
    lum = [cv2.cvtColor(cv2.resize(cv2.imread(p), (160, 90),
                                   interpolation=cv2.INTER_AREA),
                        cv2.COLOR_BGR2GRAY).mean() for p in sampled_frames]
    checks["luma_band"] = 2.0 < float(np.mean(lum)) < 60.0

    # 2. no frozen stretch (a hung layer or a dropped write)
    d = [np.abs(gray[i+1].astype(np.int16) - gray[i]).mean()
         for i in range(len(gray) - 1)]
    checks["no_freeze"] = max(run_length(lambda x: x < 0.05, d)) <= 45

    # 3. the beat grid survived the encode. This is the load-bearing assertion.
    y2, _ = librosa.load("build/out_audio.wav", sr=None, mono=True)
    env2 = librosa.onset.onset_strength(y=y2, sr=48000, hop_length=512,
                                        aggregate=np.median)
    o2 = librosa.frames_to_time(
        librosa.onset.onset_detect(onset_envelope=env2, sr=48000, hop_length=512,
                                   units='frames', delta=0.06, wait=4),
        sr=48000, hop_length=512)
    err = [min(abs(o2 - t) * 30) for t in t_beat[:64]]     # in frames
    checks["beat_sync"] = float(np.median(err)) <= 1.0

`beat_sync` re-derives onsets from the encoded, decoded audio and compares them against the grid the renderer used. A mux that shifted audio by even a few milliseconds fails it. Any `false` in `verify.json` fails the job.

**Verify:** `out/verify.json` exists and every value is `true`.

### 10. Write `out/analysis.md`

Assembled programmatically from `analysis/audio.json` and `analysis/reference.json`, regenerated every run, never hand-edited. Sections in order:

1. Source media — paths, SHA-256s, durations, decode settings.
2. Tempo — chosen BPM, candidate set (0.5× / 1× / 2×), beat-interval standard deviation, confidence flag.
3. Sections — table of `start / end / label / RMS / sub / low / mid / high / air`.
4. Reference shots — table of `index / timecode / duration / mean Lab / hex / MAD motion / pan vector`.
5. Palette comparison — reference k-means dominant colors versus the intro's `#0A0A0A` background, `#F2F0E9` text, `#C6FF3D` accent.
6. Vocabulary mapping — "what the reference does / what we do", one row per detected device, with the measured cut rate and motion MAD on the left and the corresponding scene-graph rule on the right.
7. Verification summary — the contents of `verify.json`.

**Verify:** `analysis.md` contains the exact BPM string from `audio.json`; its section-table row count equals `len(sections)`; its shot-table row count equals `len(shots)`.

### 11. Prove determinism

    - name: Render twice
      run: |
        python -m src.render --out out/brand_intro.mp4
        sha256sum out/brand_intro.mp4 | cut -d' ' -f1 > build/run1.sha
        mv out/brand_intro.mp4 build/first.mp4
        python -m src.render --out out/brand_intro.mp4
        sha256sum out/brand_intro.mp4 | cut -d' ' -f1 > build/run2.sha
        diff build/run1.sha build/run2.sha

If the hashes differ, dump `canvas` for frame 0 and frame `DURATION_FRAMES // 2` to `.npy` on both runs and diff the arrays before blaming the encoder. The cause is almost always one of four things: iteration over a `set` or over a `dict` whose insertion order varies, an unseeded RNG, a font resolved by family name instead of absolute path, or a missing `-map_metadata -1`.

**Verify:** `diff` exits 0.

### 12. Upload artifacts

`actions/upload-artifact@v4` with `retention-days: 14`, containing `out/brand_intro.mp4`, `out/analysis.md`, `out/verify.json`, `analysis/sheets/*.jpg`, `analysis/audio.json`, `analysis/reference.json`, `analysis/shots.csv`, `timeline.json`, and `render/ffmpeg.log`. Do not upload `build/` and never upload the raw frame stream.

**Verify:** download the bundle, confirm `brand_intro.mp4` plays with audio in sync at the detected beat, and confirm the contact sheets open.

---

## The top 8 pitfalls

Ordered by how often they actually destroy this specific render.

1. **ffmpeg stderr pipe deadlock.** `stderr=subprocess.PIPE` on a multi-thousand-frame render fills the 64 KB pipe buffer, ffmpeg blocks on write, your writer blocks on ffmpeg's stdin, and the job hangs until the 6-hour GitHub limit kills it. Always `stderr` to an open file handle. Same for `stdout`. Add `-nostdin` so ffmpeg never reads the runner's tty.
2. **Float time comparisons for cuts.** `if t >= cut_time` makes a cut land on frame `n` or `n+1` depending on accumulated float error, and the two runs disagree. Convert every time to an integer frame index exactly once, at timeline-compile time, and never compare times again.
3. **The 4:2:0 near-black collapse.** yuv420p stores luma on the full grid but chroma on half, so 1-px lime hairlines and mid-grey text on `#0A0A0A` lose their color and their edge. Minimum 3 px stroke, keep the background band at `#0A0A0A`–`#141414` so luma stays off the floor, and set the color tags explicitly on encode.
4. **Re-rasterizing text every frame.** 4500 frames × several 1080p text runs through FreeType is 10–40 minutes of pure waste and is the single biggest reason this job misses a 3-hour budget. Pre-rasterize each run once at 2×, downsample with `INTER_AREA`, cache the RGBA tile, and per frame only translate and composite.
5. **Fonts resolved by family name.** `ImageFont.truetype("Inter", 200)` goes through fontconfig; the runner's font set is not yours, and the same code produces different glyph widths on a different image. Use an absolute vendored path, assert its SHA-256, declare an absolute DejaVu fallback, and lay out from measured `getlength` rather than hardcoded positions so the fallback still composes.
6. **Rebuilding the MP3 decode per stage.** libsndfile and ffmpeg's MP3 decoders have different, version-dependent encoder delay. If the onset detector reads one decode and the mux uses another, audio drifts by tens of milliseconds and `beat_sync` fails. Decode once to `master.wav`, hash it, record the hash in `timeline.json`, and derive the mono analysis WAV from that same file.
7. **Nondeterministic draw order and unseeded RNG.** Iterating a `set` of layers, `random` without `seed()`, `datetime.now()` in a filename, or a missing `-map_metadata -1` (which embeds a creation timestamp) all break byte-stability. Pin `PYTHONHASHSEED=0`, `TZ=UTC`, `SOURCE_DATE_EPOCH`, seed explicitly, sort layers by an explicit integer key, and pass `-fflags +bitexact -flags:v +bitexact -flags:a +bitexact`.
8. **A green build that ships a black or frozen video.** `blackdetect` reports nothing because the background is supposed to be near-black, so the pipeline has no signal. Every failure — a layer that never enters its window, a `canvas` that was never refilled after a cut, a truncated stdin write that ffmpeg silently padded — produces a valid, playable, useless MP4. You need the luminance-band check, the inter-frame freeze check, and the re-analyzed `beat_sync` check, or you will ship garbage and call it done.

Honorable mentions that will cost you an afternoon each: `np.ndarray` not C-contiguous before `.tobytes()`; a `Popen` whose `wait()` return code is never checked; `librosa.load` defaulting to 22050 Hz so every time you compute is scaled wrong; `beat_track` with `trim=True` silently dropping the first partial beat; `scenedetect` over-segmenting a strobe-heavy reference into 400 shots; supersampling the entire 1080p canvas instead of the shape's bounding box; and PNG round-tripping "just for debugging" into the same output directory you then upload.

---

## Definition of done

- `out/brand_intro.mp4` exists: single file, H.264, yuv420p, 1920x1080, 30/1 fps, duration in `[120.0, 150.0]` s, `nb_read_frames == duration_frames`, both stream `start_time`s `0.000000`.
- The deliverable's audio is AAC 48 kHz stereo derived from the byte-identical `build/master.wav` whose SHA-256 is recorded in `timeline.json`.
- `timeline.json` contains only integer frame indices; its cut list is strictly increasing with a minimum 6-frame gap; every cut frame equals `int(round(t_onset * 30))` for a detected onset.
- Every layer passes the compile-time straddle assertion, so no layer crosses a cut boundary unless explicitly allowed.
- Two consecutive renders of the same commit produce identical SHA-256 for `out/brand_intro.mp4`.
- Peak RSS during render is under 2 GB and no PNG or JPEG frame file is written anywhere in the pipeline.
- `out/verify.json` has every check `true`, including `beat_sync` with median error ≤ 1 frame against re-analyzed decoded audio.
- `out/analysis.md` and at least one contact-sheet JPG under `analysis/sheets/` are present, and the markdown's tempo, section count, and shot count match `analysis/audio.json` and `analysis/shots.csv` exactly.
- The render log records the absolute font path actually used, and that path is either the pinned vendored TTF or the declared absolute DejaVu fallback.
- No geometry in the scene graph is thinner than 3 px.
- The workflow completes within `timeout-minutes: 170` and the artifact bundle downloads intact.