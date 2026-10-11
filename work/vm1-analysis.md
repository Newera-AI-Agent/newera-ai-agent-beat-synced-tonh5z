# VM JOB 1 — Media acquisition, deep analysis, and asset vendoring

## Goal

By the end of this job, the repo contains REAL, machine-generated analysis of both source
media files plus the vendored fonts needed later. No rendering of the final film in this job.

## Inputs (download these yourself)

- Reference video: `https://d1oupeiobkpcny.cloudfront.net/assets/dashboard/materials/2026/09/28/9e16cfc6dbd4e33982113ccbb237cbdc67e723f6df1378a05701a95cf65ec5dc.mp4`
- BGM audio: `https://tmpfiles.org/wFAk6KtKsGFz/ptasinskirjpasin-lifeforce.mp3`
  - Fallback: `https://files.catbox.moe/8wi7fw.mp3`
  - Note: tmpfiles.org serves a landing page; use the direct form `https://tmpfiles.org/dl/wFAk6KtKsGFz/ptasinskirjpasin-lifeforce.mp3`

Store them as `assets/source/reference_manus.mp4` and `assets/source/life_force.mp3`.
Record `sha256sum` of both into `analysis/sources.json`.

## Deliverables (exact paths)

1. `analysis/sources.json` — urls, byte sizes, sha256, ffprobe metadata for both files.
2. `analysis/audio.json` — the full BGM analysis (schema below).
3. `analysis/audio.md` — a written, batch-by-batch audio report (see "Audio report" below).
4. `analysis/reference.json` + `analysis/reference.md` — shot-level analysis of the reference video.
5. `analysis/sheets/*.jpg` — contact sheets (see below).
6. `assets/fonts/` — at least 2 vendored open-source sans-serif TTFs, absolute-path usable.
   Recommended (download from Google Fonts GitHub raw):
   - `Inter` (variable or static Medium/SemiBold/Bold) — primary display + body
   - `Space Grotesk` or `JetBrains Mono` — secondary / technical accents
   Also write `assets/fonts/FONTS.lock` listing filename + sha256 + source URL + license.
7. `analysis/spectrum.png` — matplotlib figure: waveform, onset envelope, log-mel spectrogram,
   per-band energy curves, detected beats and section boundaries drawn as vertical lines.
8. `analysis/waveform.md` — plain-language description of the track's structure and dynamics.

## Audio analysis — required schema (`analysis/audio.json`)

Decode exactly once, as the skill says:

```bash
ffmpeg -hide_banner -nostdin -y -loglevel error -i assets/source/life_force.mp3 \
  -vn -ac 2 -ar 48000 -c:a pcm_s24le build/master.wav
ffmpeg -hide_banner -nostdin -y -loglevel error -i build/master.wav \
  -vn -ac 1 -ar 48000 -c:a pcm_f32le build/analysis.wav
```

Use hop_length 512, n_fft 2048, sr 48000. Required fields:

- `duration_s`, `sample_rate`, `master_wav_sha256`
- `bpm`, `bpm_confidence` (std of beat intervals in seconds), `bpm_candidates` (0.5x, 1x, 2x)
- `beat_times` (float seconds), `onset_times` (float seconds) — full lists, no truncation
- `band_energy` with keys `sub`(20-120) `low`(120-500) `mid`(500-2000) `high`(2000-8000) `air`(8000-16000),
  each a per-hop float array over the whole track
- `rms` per-hop array over the whole track
- `spectral_centroid` per-hop array
- `sections`: list of `{start_s, end_s, label, rms_mean, band_means{...}}` where label is one of
  `intro|build|drop|breakdown|peak|outro`, derived from novelty peaks snapped to the nearest beat
- `suggested_duration_s`: pick a musically final moment in [120, 150] that lands on a section
  boundary or a downbeat; state the reason in `suggested_duration_reason`.

## Audio report — `analysis/audio.md`

Written in BATCHES of ~15 seconds of the track. For each batch report: timecode range, RMS level,
which bands dominate, what instrument/texture the spectrum implies, density of onsets (events/sec),
and a one-line "what a cut should do here" recommendation (e.g. "rapid 4-6 frame cuts, type swaps
on every 8th note", "hold, slow push, single line of type"). End with a global summary table:
BPM, total onsets, onsets/sec by section, dynamic range, and the strongest 10 transient times.

## Reference video analysis

- Scene detect with `scenedetect detect-content --threshold 27 --min-scene-len 15`, export `analysis/shots.csv`.
- For EVERY shot, extract the middle frame and compute: start/end timecode, duration,
  mean Lab color, dominant hex (k-means k=5 over shot means), motion MAD between first and last
  frame of the shot, and the phase-correlate pan vector.
- `analysis/reference.md` must contain: shot count, median shot length, cut-rate histogram,
  palette table with hex codes, motion distribution, and a "visual vocabulary" section naming the
  3-5 recurring compositional devices actually observed (e.g. centered single-line type on black,
  thin horizontal rule wipes, counter/digit roll, grid of small squares, letter-by-letter reveal).
- Also report the reference video's own audio characteristics (does it have music? speech? silence?)
  using the same band analysis, briefly.

## Contact sheets (required, and they will be inspected by a human)

- `analysis/sheets/grid_XX.jpg` — tiles of `fps=1/4, scale=384:-1, tile=5x4` one per 40 s.
- `analysis/sheets/shots_XX.jpg` — labeled per-shot sheets, 4x3 grid, each tile annotated with
  `#NNN  t=MM:SS.mmm` in off-white on near-black. Generate enough sheets to cover every shot.
- Also produce `analysis/sheets/first_20s.jpg` — 10 tiles at 2 s spacing from the opening, and
  `analysis/sheets/last_20s.jpg` similarly for the ending. These bracket the parts that matter most.

## Environment

- `ubuntu-24.04`, ffmpeg 6.x, Python 3.11, CPU only, no GPU.
- Pin: `numpy==1.26.4 scipy==1.13.1 librosa==0.10.2.post1 numba==0.60.0 llvmlite==0.43.0
  soundfile==0.12.1 Pillow==10.4.0 opencv-python-headless==4.10.0.84 scenedetect==0.6.5
  jsonschema==4.23.0 matplotlib==3.9.2`
- Env: `TZ=UTC PYTHONHASHSEED=0 LC_ALL=C.UTF-8`.
- Write `requirements.lock.txt` with those exact pins.

## Hard rules

- Never decode the MP3 twice. Everything reads `build/master.wav` or `build/analysis.wav`.
- Never pipe ffmpeg stderr to a pipe; redirect to a file handle, always pass `-nostdin -loglevel error`.
- All JSON written with `sort_keys=True, indent=2`, and every float rounded to 6 decimals.
- Every number in the markdown reports must be generated from the JSON, never typed by hand.
- If a download fails, retry once after 10 s, then try the fallback URL, then fail loudly with the
  exact HTTP status and URL in the log. Do not silently substitute a different asset.

## Acceptance (must all be true when you finish)

- `assets/source/reference_manus.mp4` and `assets/source/life_force.mp3` exist with recorded sha256.
- `ffprobe` on the reference reports width 1920 or similar and duration > 60 s.
- `analysis/audio.json` parses, has > 100 beats, > 300 onsets, and 3+ sections.
- `analysis/shots.csv` has >= 5 rows; every sheet JPG exists and is > 1000 px wide.
- `assets/fonts/` has >= 2 TTF files that PIL can load at size 64 without error.
- `analysis/spectrum.png` exists and is > 1000 px wide.
- `python -c "import json;json.load(open('analysis/audio.json'))"` exits 0.
- Print a final `SUMMARY` block with: bpm, bpm_confidence, duration, section count, onset count,
  shot count, median shot length, palette hexes, and every output path with its byte size.
