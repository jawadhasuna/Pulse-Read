# Pulse Read

**Generate radar pulses and radio modulations, look at them properly, then train a
classifier — all inside your browser.**

### 🔗 Live site: <https://pulsereads.vercel.app>

No server, no Python, no libraries. The signals, the feature extraction, the FFT and
the Random Forest are all written from scratch and run on your machine.

---

## What it does

**Generate** a signal from one of two catalogues:

- **Radar pulses** — six airborne fire-control radars (AN/APG-77, APG-81, APG-79,
  AWG-9, APQ-181, APG-63) with their published pulse-width and PRF ranges, and their
  intra-pulse modulations: linear chirp, non-linear chirp, stepped frequency,
  13-bit Barker code, polyphase, and low-probability-of-intercept waveforms.
- **Radio modulations** — twelve of them: AM, FM, SSB, BPSK, QPSK, 8PSK, 16-QAM,
  64-QAM, FSK, OFDM, frequency hopping and direct-sequence spread spectrum.

**Look at it** four ways at once: amplitude over time, frequency spectrum,
spectrogram, and the I/Q plane.

**Measure it** — 23 numbers pulled out of the samples, each explained in plain words.

**Train on it** — build a dataset, grow a Random Forest, and get a confusion matrix
plus a ranking of which measurements the forest actually leaned on.

**Test it** — throw fresh signals at the trained model and watch it succeed or fail.

---

## The honest numbers

Measured on this implementation, 70/30 split, 25 examples per class:

| Task | Accuracy |
|---|---|
| **Radio modulation** (12 classes) | **~97%** |
| **Radar type** (6 classes) | **~63%** |

Those two numbers are very different, and the gap is the most interesting thing here.

**Why radio is easy.** Modulations differ in ways that survive noise. Raising a
unit-amplitude M-PSK signal to its Mth power collapses every symbol onto one point,
so the magnitude of that average jumps to nearly 1 — and stays near 0 for everything
else. Measured here:

| | 2nd power | 4th power | 8th power |
|---|---|---|---|
| BPSK | **1.00** | 1.00 | 0.95 |
| QPSK | 0.01 | **0.99** | 0.95 |
| 8PSK | 0.10 | 0.03 | **0.95** |
| 16-QAM | 0.06 | 0.36 | 0.06 |
| 64-QAM | 0.02 | 0.11 | 0.01 |

That one trick takes the classifier from 92% to 97%.

**Why radar is hard.** Look at the published ranges: the APG-77 fires 0.5–5 µs pulses
at 2000–10000 Hz; the APG-79 fires 0.8–8 µs at 1200–12000 Hz. They overlap almost
entirely. A single observation frequently *cannot* tell you which aircraft you are
looking at, because both radars can legitimately produce the same pulse. 63% is the
honest ceiling for one look — not a broken model.

---

## About that 100% in the original

The Python tool this came from reported near-perfect radar classification. It was
reading the answer off its own input.

Its feature vector included `beam_width`, `peak_power`, `polarization`, `scan_pattern`
and `classification` — every one of them copied straight out of the ground-truth table
for the very radar being predicted. Beam width and peak power cannot be measured from
a received pulse; they *are* the label, wearing a different hat. That is textbook
label leakage.

Reproduced here, with the same six radars and the same forest:

| Feature set | Accuracy |
|---|---|
| Only what can be measured from the signal | **63%** |
| Plus the copied spec-sheet values | **100%** |

The **"Also feed it the spec sheet"** tick-box on the site does exactly this, so you
can watch the accuracy leap and see the leaked features shoot to the top of the
importance chart in red. It is the most useful thing this project demonstrates:
a model scoring 100% is usually a bug, not a triumph.

---

## One correction to the physics

Both Python programs generated an 8–12 GHz carrier sampled at 100 MHz. That breaks
the Nyquist limit by a factor of about 200 — what came out was aliasing artefacts,
not a radar pulse.

Pulse Read works at **complex baseband**, which is what a real receiver produces after
mixing the carrier down. Nothing identifying is lost: pulse width, PRF, bandwidth,
chirp shape and phase coding all live in the envelope.

---

## How it works under the hood

| Piece | Method |
|---|---|
| FFT | Iterative radix-2, in place, written here |
| Spectrogram | Hann-windowed STFT drawn to an ImageData buffer |
| Features | 23 measurements: envelope statistics, instantaneous frequency, spectral shape, centre-frequency drift, M-th power moments, and pulse-domain timing |
| Classifier | Random Forest — CART trees on Gini impurity, bootstrap bagging, √n features per split, Gini-decrease importance |
| Split | Stratified 70/30, so every class appears in both halves |
| Speed | 60 trees on ~300 examples trains in well under 200 ms |

Dataset building runs in animation-frame slices so the page never freezes, and the
whole dataset exports to CSV if you want to take it to scikit-learn.

---

## Running it locally

Single HTML file, no dependencies, no build step.

```bash
git clone https://github.com/jawadhasuna/Pulse-Read
cd Pulse-Read
python -m http.server 8000
```

Then open <http://localhost:8000>. Opening `index.html` directly works too.

## Deploying

Any static host. This repo is set up for [Vercel](https://vercel.com) — import and deploy.

---

## Origin

Merged from two Python + Tkinter prototypes that were the same program twice over:
`rdr_classifier4.py` (radar pulses) and `radio_classifier1.py` (RF modulations). Both
used NumPy, SciPy, scikit-learn and SQLite, and both had the identical three-tab
layout — generate, train, test. One codebase with a family switch replaces both.

**What did not cross over:** the SQLite store and the MLP classifier. A page with no
server has nowhere to keep a database, and a second model added no insight next to
the forest's feature-importance output.

## Licence

MIT
