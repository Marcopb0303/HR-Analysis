# fhr-analysis — Repo Guide

This document is a map of the repository: what each folder is for, what the
scripts inside it do (at a glance, not line-by-line), and the different
end-to-end workflows ("pipelines") you can run. Read this alongside
[README.md](README.md), which has the quick-start setup commands.

## What this project does

The project extracts **fetal heart rate (FHR)** from acoustic sensor
recordings taken on a pregnant patient's abdomen ("Banner" fiber-optic
sensors), and scores how accurate the extraction is against a "source of
truth" (SOT) — a microphone or PPG (pulse-oximeter) recording taken at the
same time as a separate reference.

The hard part is that the abdomen recording is a *mixture*: fetal heart
sounds, maternal heart sounds, lung/breathing, and noise are all picked up
together. So most of the code is about **source separation** (pulling the
fetal heartbeat out of that mixture) and then **beat detection** (finding
individual heartbeat timestamps in the separated signal), followed by
**evaluation** (comparing detected beats/heart-rate-over-time against the
SOT).

Several separation methods are implemented, and each is a different
"pipeline" you can choose to run for a given recording:

| Method | What it is |
| --- | --- |
| **ICA** | Independent Component Analysis — classic blind source separation |
| **MNMF** | Multichannel Non-negative Matrix Factorization |
| **MLCMED** | An NMF/median-filter based separation method |
| **NMCF** | Non-negative Matrix Co-Factorization |
| **NeoSSNet** | A pretrained deep-learning source-separation model (submodule), optionally fine-tuned on this project's own recordings |
| **FUNet** | A custom-trained deep-learning model that predicts *beat activity* directly (skips separating a clean waveform) |
| **Raw bandpass** | No separation at all — just bandpass-filter the raw abdomen fiber into the fetal acoustic band |

FUNet and NeoSSNet are the two actively-developed, best-performing methods;
ICA/MNMF/MLCMED/NMCF are earlier/classical baselines kept for comparison.

---

## Top-level layout

```
fhr-analysis/
├── setup.sh              Bootstraps the Python venv + submodules (see README)
├── requirements.txt       Python deps for the main analysis code (src/)
├── bin/                   Standalone data-collection apps (own venvs, PyQt GUIs)
├── lib/                   The separation/detection models: submodule + training code + checkpoints
└── src/                   The main analysis library, CLI tools, and the beat-marking web app
```

Two directories referenced throughout are **gitignored** (not in version
control, must exist locally or be fetched separately):
- `Banner_data/` — the actual patient recordings (fiber `.npy` bundles,
  microphone `.wav`, hand-marked beat `.npy` files). Nothing runs without this.
- `.out/` — cache and output files (plots, evaluation results) written by
  pipeline runs, keyed by patient/method/window.

---

## `bin/` — Data collection apps

Two standalone PyQt5 desktop apps used to **record** sensor data in the field.
These are separate from the analysis code (each has its own
`requirements.txt`) and aren't imported by anything in `src/`.

```
bin/
├── record/                      Basic multi-channel PicoScope recorder
│   ├── src/main.py              PyQt5 GUI: live scope view, start/stop recording to .npy
│   └── lib/pico/                Git submodule: PicoScope Python SDK wrapper
└── record_with_realtime_tracking/
    └── src/
        ├── main.py              GUI entry point (adds BLE + live HR tracking on top of record/)
        ├── hr_analysis.py       Real-time heart-rate estimation while recording
        ├── hr_panel.py          Live HR display widget
        └── epoch_axis.py        Time-axis helper for the live plot
```

`record_with_realtime_tracking` additionally pulls in a Bluetooth reference
device (`bleak`) and live audio (`pyaudio`) so a maternal/reference HR trace
can be watched in real time while recording — useful for confirming sensor
placement is good before a session ends.

---

## `lib/` — Models: pretrained code, fine-tuning, and checkpoints

Everything needed to separate/detect the fetal heartbeat via deep learning
lives here: the model architectures, their training scripts, and the trained
weights (checkpoints), versioned as you iterate.

```
lib/
├── neossnet/            Git submodule (external repo, not vendored):
│                         base pretrained NeoSSNet source-separation model + its
│                         own training code. Must be fetched via
│                         `git submodule update --init --recursive`.
│
├── tune-ssnet/           Fine-tunes NeoSSNet on this project's own recordings
│   ├── src/main.py       Fine-tuning entry point (imports the submodule's model/train code)
│   ├── fetal-tune-config.yaml     Config: fine-tune towards the FETAL source
│   ├── maternal-tune-config.yaml  Config: fine-tune towards the MATERNAL source
│   ├── training_clips.yaml        Which recording windows/fibers to build snippets from
│   ├── generate_training_snippets.sh  Builds the training snippet set from training_clips.yaml
│   └── models/                    One folder per fine-tuning run (tuned-model-v1 … v13,
│                                   maternal-tuned-model-v2), each with model_best.pt +
│                                   model.yaml (architecture config used for that checkpoint)
│
├── funet/                A custom model (FUNet) trained from scratch to output a
│   │                     per-frame "beat activity" signal instead of a separated waveform
│   ├── src/
│   │   ├── model.py       FUNet architecture (U-Net-style conv encoder/decoder)
│   │   ├── train.py       Training loop
│   │   ├── main.py        Training entry point (wires config → data → model → loss → train)
│   │   ├── data.py        Dataset loading for FUNet's snippet format
│   │   ├── augment.py     Data augmentation for training
│   │   ├── loss.py        Loss functions (SNR, correlation, MSE, KL-div variants)
│   │   ├── config.py      Config file schema/loader
│   │   └── inference.py   Runs a trained FUNet checkpoint on a raw waveform
│   ├── fetal-config.yaml           Default training config (architecture + hyperparameters)
│   ├── training_clips.yaml         Which recording windows/fibers to build snippets from
│   ├── generate_training_snippets.sh  Builds the training snippet set
│   ├── rfp_train/                 "Rough-then-fine pass" alternative training recipe:
│   │   ├── rough-pass-config.yaml / rough_pass_training_clips.yaml   stage 1: broad training
│   │   ├── fine-pass-config.yaml / fine_pass_training_clips.yaml     stage 2: fine-tune stage 1's output
│   │   └── intermediary/v1/, v2/  checkpoints saved between the two stages
│   └── models/                    One folder per training run (funet-v1 … v32, plus a
│                                   "(CONTROL)" baseline), each with fetal-config.yaml +
│                                   model_best.pt / model_last.pt / training_curves.png
│
└── beats/                Hand-marked reference beat timestamps (.npy) for a few
                           patients, exported from the beat-marking app (src/beat_app);
                           used as ground truth outside the normal Banner_data layout.
```

**Model versioning convention:** every training run gets its own numbered
folder under `models/` (`funet-v32`, `tuned-model-v13`, …) so old checkpoints
are never overwritten and results stay comparable across iterations. Which
checkpoint is actually *used* by the analysis pipelines is selected in
`src/constants.py` (`FUNET_MODEL`, `FETAL_MODEL`, etc. — see below).

---

## `src/` — Main analysis library, CLI tools, and beat-marking app

```
src/
├── constants.py           Shared config: paths, which model checkpoint is
│                          "active", sample rates, acoustic frequency bands, BPM ranges
├── analyze/               The core library (imported as `from analyze.X import ...`)
├── bin/                    Standalone CLI utilities (data prep, plotting, SNR)
└── beat_app/               Local web app for viewing/hand-correcting beat marks
```

### `src/constants.py`

The single place that picks **which trained model checkpoint each pipeline
uses** (`FUNET_MODEL = "funet-v32"`, `FETAL_MODEL = "tuned-model-v13"`, etc.),
plus shared physical constants: acoustic frequency bands for filtering
(maternal vs. fetal), BPM ranges used to sanity-check detected beats, device
sample rates, and per-patient data file names (`MIC_BEATS_FILE`,
`NST_DRIFT_LOG_FILE`, etc.). **To switch which model version a pipeline runs
against, edit this file.**

### `src/analyze/` — the core library

```
src/analyze/
├── data.py                 Audio / FiberData / FiberPair containers; loads
│                            raw recordings from a Banner_data patient folder
├── filters.py               Bandpass / notch filter stage builders
├── pipeline.py              Pipeline class: chains processing "stages" with
│                            automatic content-hash caching (re-running a
│                            pipeline skips stages whose inputs haven't changed)
├── main.py                  Entry point: wires stages together into full
│                            end-to-end runs (see "Pipelines" below)
│
│   -- Source separation methods (pick one per pipeline) --
├── ica.py                   ICA-based separation
├── mnmf.py                  Multichannel NMF separation
├── mlcmed.py                 MLCMED separation
├── nmcf.py                  NMCF separation
├── neossnet.py               NeoSSNet deep-learning separation pipeline
├── funet.py                  FUNet beat-activity model pipeline
│
├── anc.py                    Adaptive noise cancellation (NLMS / Kalman) —
│                            used after separation to cancel remaining maternal contamination
├── imf_select.py             Selects which decomposition component is "the fetal one"
├── wavelet_denoise.py         Optional wavelet-based denoising stage
├── clean_data_template.py     Template/scratch script for one-off data cleaning
│
├── hr/                        Beat detection ("find heartbeat timestamps in a waveform")
│   ├── detect.py … detect_v8.py   Eight detector versions (v1 through v8), each
│   │                               a different peak-picking/segmentation strategy —
│   │                               later versions generally supersede earlier ones,
│   │                               kept side-by-side for comparison. Each file's
│   │                               docstring explains why that version exists and
│   │                               how it differs from its predecessor.
│   ├── classify.py             Given several candidate separated sources, picks
│   │                            which one is "the fetal source"
│   └── utils.py                 Shared beat-detection helpers
│
├── sot.py                     Loads the "source of truth" (mic or PPG) and
│                              detects its reference beats
├── drift.py                    Corrects NST/mic beat timestamps for dropped-sample
│                              clock drift, given a per-patient drift log (see
│                              "Correcting NST clock drift" below)
├── evaluate.py / evaluate_v2.py / evaluate_v3.py
│                              Score detected beats against the SOT (precision/
│                              recall/F1, HR-over-time comparison plots) — three
│                              versions with progressively better lag-alignment
│                              and windowing logic; v3 is the most current
├── plot_hr.py                 Plots heart-rate-over-time and beat/peak overlays
│                              (includes the drift-corrected comparison plot)
└── util.py                    Misc shared helpers (e.g. invoking NeoSSNet)
```

### `src/bin/` — standalone CLI utilities

Each script inserts `src/` onto `sys.path` itself, so these can be run
directly (`python3 src/bin/<script>.py`) without extra setup, from anywhere.
Most have a matching `.sh` wrapper that activates the venv first.

```
src/bin/
├── generate_training_snippets.py   Builds paired fetal-HR + SOT training
│                                    snippet datasets (used by both funet/
│                                    and tune-ssnet/ to prepare model training data)
├── pico2data.py (+ .sh)             Converts raw PicoScope CSV exports into
│                                    this project's .npy / .wav data format
├── concat.py                        Concatenates two .npy recordings into one
├── change_start.py                  Prepends silence to a .npy recording to
│                                    correct for a recording that started late
├── s1s2_diagnostic.py                Diagnostic script checking whether FUNet
│                                    timing errors are S1/S2 heart-sound swaps
├── analyze/
│   ├── analyze_waveforms.py (+ .sh)  Plots every waveform in a Banner_data
│   │                                folder (sanity-check raw recordings)
│   └── plot_clips.py (+ .sh)         Plots the fiber/SOT clips referenced by a
│                                    training-snippet yaml (sanity-check training data)
├── peak_det/
│   └── plot_peak_detectors.py (+ .sh)  Runs and compares every hr/ beat
│                                      detector side-by-side on one patient
└── snr/
    └── calculate_snr.py (+ .sh)      Computes signal-to-noise ratio for a recording
```

### `src/beat_app/` — the beat-marking web app

A local tool for creating/correcting the hand-marked "ground truth" beat
timestamps used elsewhere (e.g. `lib/beats/*.npy`, `MIC_BEATS_FILE`). Load a
WAV, run any `hr/*_beat_detector` on it as a starting point, then manually
add/move/delete beat markers on a waveform canvas, and export to `.yaml` or
`.npy`. See [src/beat_app/README.md](src/beat_app/README.md) for the full
workflow and keyboard/mouse controls.

```
src/beat_app/
├── server.py            Stdlib HTTP server: routes, sessions, background detect jobs
├── detectors.py          Auto-discovers hr/*_beat_detector functions for the dropdown
├── audio_io.py            WAV loading; YAML/.npy beat (de)serialization
└── static/                 Frontend: index.html, style.css, app.js (Canvas UI), hr.html (pop-out HR graph)
```

Run it with:
```bash
python3 src/beat_app/server.py            # opens http://127.0.0.1:8000
```

---

## Setup

```bash
./setup.sh
source .venv/bin/activate
```

`main.py` and other `src/analyze/` entry points import `analyze.*` / `constants`
as top-level packages, so `src/` needs to be on `PYTHONPATH` — either
`PYTHONPATH=src python src/analyze/main.py`, or run from inside `src/`.

This creates `.venv`, installs `requirements.txt`, and fetches the
`lib/neossnet` (and `bin/record/lib/pico`) git submodules. `bin/record*`
apps have their own separate `requirements.txt` — install those into the
same or a separate venv only if you need the recording GUIs.

---

## Running an analysis pipeline

The main way to run an end-to-end analysis is `src/analyze/main.py`. It
defines one function per separation method (`ica()`, `run_mlcmed_pipeline()`,
`run_nmcf_pipeline()`, `run_mnmf_pipeline()`, `run_raw_bandpass()`, plus
`run_funet_pipeline(...)` / `run_neossnet_pipeline(...)` imported from
`analyze.funet` / `analyze.neossnet`). At the bottom of the file, under
`if __name__ == '__main__':`, the calls you want are left un-commented (most
are commented out — that's the mechanism for choosing which pipeline(s) run).

```bash
# 1. Pick the patient + time window at the top of the file:
#      PATIENT = "PT12_2"
#      WINDOW = 0, 340
# 2. Uncomment the pipeline call(s) you want at the bottom of the file.
# 3. Run it:
PYTHONPATH=src python src/analyze/main.py
```

Every pipeline follows the same shape:
1. Load the SOT (mic/PPG) reference and detect its reference beats.
2. Load the raw fiber recording, window it to the time range of interest, and
   bandpass/notch filter it.
3. Run the chosen separation method (or skip straight to bandpass-only).
4. Detect beats in the separated fetal source with one of the `hr/detect_v*`
   detectors.
5. Evaluate detected beats against the SOT and write plots/scores to
   `.out/<patient>/<method>/`.

Each `Pipeline` caches intermediate stage outputs by content hash under an
`out/.cache*` folder, so re-running with the same inputs is fast — only
stages whose upstream inputs actually changed are recomputed.

### Choosing a pipeline / model version

- **Which separation method** → which function you call in `main.py`
  (`run_funet_pipeline`, `run_neossnet_pipeline`, `run_mnmf_pipeline`, etc).
- **Which trained checkpoint** that method uses → edit `src/constants.py`
  (`FUNET_MODEL`, `FETAL_MODEL`, `MATERNAL_MODEL_PATH`, `NEOSSNET_MODEL_PATH`).
- **Which patient recording / time window** → the `PATIENT` / `WINDOW`
  constants at the top of `main.py`.

### How FUNet (v24) and the NST detector (v7) interact

`main.py`'s current default run is `run_funet_pipeline` on `funet-v24` with
`v7_beat_detector` (`src/analyze/hr/detect_v7.py`, a duration-dependent HMM
Springer/Schmidt-style heart-sound segmenter). Both the NST/mic side and the
fiber side of that pipeline call `v7_beat_detector`, but on two structurally
different signals — worth spelling out:

1. **NST/mic side** — `sot_beats(v7_beat_detector, out_path)` runs v7 on the
   real microphone recording, band-limited to the fetal acoustic band. Its
   output (`sot.mic_beats`) is the reference every fiber method is scored
   against.
2. **Fiber side** — `use_funet` (in `src/analyze/funet.py`) stacks the 5
   abdomen fibers `1B, 2A, 2B, 2C, 2D` (funet-v24 is a 5-channel model — the
   fiber count passed to `run_funet_pipeline` must match `FUNET_MODEL`'s
   `config.model.channels`) into the FUNet U-Net (`lib/funet/src/model.py`).
   FUNet's output isn't a cleaned-up *acoustic* waveform, it's a single-channel
   **beat-activity envelope** — already shaped like a clean pulse train, one
   bump per heartbeat, with the acoustic mixture stripped away. `fiber_beats`
   then runs that *same* `v7_beat_detector` on this activity envelope (as the
   "fetal" source) and, separately, on the raw maternal-band-filtered chest
   fiber (as the "maternal" source).

So v7 plays double duty: once on genuine acoustic audio (the mic), once on a
model-generated activity envelope with very different statistics. That's a
deliberate choice — using one detector on both sides keeps the comparison in
`hr_comparison.png` apples-to-apples, so an HR mismatch reflects FUNet's
separation quality rather than a detector mismatch — but it also means v7's
tuning (HMM state-duration priors, etc.) was designed around real heart-sound
acoustics, not the sharper, denser envelope FUNet actually outputs.

`funet.py` also defines `funet_beats()`, a peak-picker built specifically for
FUNet's activity envelope (adaptive-threshold `find_peaks`, no HMM), but
`run_funet_pipeline` does **not** call it — it's available if you want to
compare v7's HMM-based picks against a simpler envelope-native peak-picker on
the fiber side.

---

## Correcting NST clock drift

The NST/mic recording's timestamps are computed as `sample_index / mic_fs`.
If the recording drops samples partway through, that nominal clock
increasingly falls behind real elapsed time — every beat timestamp
`detect_v7` (or any detector) reports on the NST gets progressively later in
real terms than the nominal time says. `analyze.drift` corrects for this
given a log of when dropouts happened and how much time each one cost.

**Format** — a CSV named `nst_dropouts.csv` (constant: `NST_DRIFT_LOG_FILE` in
`src/constants.py`), placed in the patient folder next to `microphone.wav`:

```csv
time_s,seconds_lost
12.340,0.008
45.900,0.015
201.220,0.006
```

- `time_s` — nominal NST time (same clock as `microphone.wav`) the dropout
  was detected at.
- `seconds_lost` — how much real time that dropout cost, in seconds (convert
  from a sample count using the mic's sample rate if that's what you have).

**What happens with it:** for each NST beat time, `drift.correct_drift` sums
`seconds_lost` over every dropout at or before that beat's nominal time, and
shifts the beat later by that cumulative amount. `run_funet_pipeline` and
`run_neossnet_pipeline` both run this automatically and write a **second**,
separate plot — `hr_comparison_corrected.png` — alongside the original
`hr_comparison.png`, so corrected vs. uncorrected can be compared directly. If
`nst_dropouts.csv` doesn't exist for a patient, the corrected plot is a no-op
copy of the original.

---

## Training / fine-tuning a model

Both deep-learning methods follow the same three-step recipe: **build
training snippets → train → point `constants.py` at the new checkpoint.**

### FUNet (train from scratch)

```bash
# 1. Build training snippets from lib/funet/training_clips.yaml
./lib/funet/generate_training_snippets.sh

# 2. Train
.venv/bin/python lib/funet/src/main.py lib/funet/fetal-config.yaml
```
Checkpoints land in the config's `model_dir` (convention: a new
`lib/funet/models/funet-vN/` per run). An alternative "rough pass, then fine
pass" recipe lives in `lib/funet/rfp_train/` for staged training.

### NeoSSNet fine-tuning

```bash
# 1. Build training snippets from lib/tune-ssnet/training_clips.yaml
./lib/tune-ssnet/generate_training_snippets.sh

# 2. Fine-tune towards the fetal source (or maternal-tune-config.yaml for the maternal source)
.venv/bin/python lib/tune-ssnet/src/main.py lib/tune-ssnet/fetal-tune-config.yaml
```
This starts from the pretrained `lib/neossnet` submodule model and fine-tunes
on this project's own recordings. Checkpoints land under
`lib/tune-ssnet/models/tuned-model-vN/`.

After either training run, update `FUNET_MODEL` / `FETAL_MODEL` in
`src/constants.py` to point at the new version so the analysis pipelines
pick it up.

---

## Other common workflows

- **Bring in a new PicoScope recording:**
  `.venv/bin/python src/bin/pico2data.py <export.csv> --out <dest>` to convert
  it into the `.npy`/`.wav` layout the rest of the code expects.
- **Sanity-check a raw recording:**
  `src/bin/analyze/analyze_waveforms.sh <patient_dir>` to plot every channel.
- **Compare beat detectors on one recording:**
  `src/bin/peak_det/plot_peak_detectors.sh <patient>` to run all of
  `hr/detect_v1`…`v8` side-by-side against the hand-marked SOT.
- **Check signal quality:** `src/bin/snr/calculate_snr.sh <patient_dir>`.
- **Create/correct ground-truth beat marks:** `python3 src/beat_app/server.py`.
- **Fix a recording that started late, or splice two recordings together:**
  `src/bin/change_start.py`, `src/bin/concat.py`.

---

## Data layout expected under `Banner_data/`

Each patient/session is a folder containing (names driven by
`src/constants.py`):
- `ps4000.npy` — chest sensor bundle (`FIBER_BUNDLE_A`)
- `ps3000a.npy` — abdomen sensor bundle, fibers `1B`/`2A`/`2B`/`2C`/`2D`
  (`FIBER_BUNDLE_B`)
- `microphone.wav` — reference microphone recording (SOT)
- `mic_beats.npy` — hand-marked fetal beat times for that mic recording
  (from the beat-marking app)
- `pvs.npy` — PPG-derived source-of-truth data, where available
- `nst_dropouts.csv` *(optional)* — dropped-sample drift log for the NST/mic
  recording; see "Correcting NST clock drift" above

This directory is gitignored; you need real patient data locally (or
generated synthetic/test recordings) for any pipeline to run.
