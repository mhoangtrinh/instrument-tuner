# Instrument Tuner: developer notes

Technical details for working on the tuner. The user-facing overview is in [README.md](README.md).

The app started as a rewrite of an earlier MATLAB App Designer tuner (kept in [`matlab/`](matlab/) for reference).

## Reference sounds

Every recording is pitch-corrected: its pitch is tracked over time and flattened exactly onto the note, which also removes the violin's vibrato. Each candidate is then measured with the tuner's own engine, and only the steadiest is kept (spread ≤ ±4 cents). Its small remaining offset is stored and corrected at playback. The app shifts a recording by at most two semitones up, or a few down, to reach any string; otherwise it uses the modelled plucked string ([`src/audio/pluck.ts`](src/audio/pluck.ts)). Rebuild the samples with:

```bash
python3 scripts/make-samples.py && npx tsx scripts/pick-samples.ts
```

## Troubleshooting clips

Settings → *Save the last 10 seconds* writes a WAV with the audio plus a `tunr` chunk holding the settings and what the tuner showed. It can be replayed through the [session tests](uxtest/).

## How it works

Microphone audio passes through a low-pass pre-filter and then into a YIN pitch detector, which gives the frequency and a confidence value. The app compares that frequency to the nearest string or note and shows the deviation in cents.

| Piece | File |
|---|---|
| YIN pitch detector | [`src/pitch/yin.ts`](src/pitch/yin.ts) |
| Streaming low-pass pre-filter | [`src/pitch/filter.ts`](src/pitch/filter.ts) |
| Detector settings per frequency range | [`src/pitch/config.ts`](src/pitch/config.ts) |
| Note math (equal temperament, adjustable A4) | [`src/tuning.ts`](src/tuning.ts) |
| Instruments and preset tunings (data only) | [`src/instruments.ts`](src/instruments.ts) |
| Tuning selection and custom tunings (pure logic) | [`src/tunings.ts`](src/tunings.ts) |
| Original MATLAB algorithm, ported as-is for comparison | [`src/pitch/legacy.ts`](src/pitch/legacy.ts) |
| Live tuner logic: samples in, readings out (no DOM) | [`src/engine.ts`](src/engine.ts) |
| Microphone capture (AudioWorklet) | [`src/audio/mic.ts`](src/audio/mic.ts) |
| Reference tone | [`src/audio/reference-tone.ts`](src/audio/reference-tone.ts) |
| UI wiring | [`src/main.ts`](src/main.ts), [`index.html`](index.html), [`src/styles.css`](src/styles.css) |
| All user-facing text (for translations) | [`src/i18n.ts`](src/i18n.ts) |
| Offline support (service worker registration, updates) | [`src/pwa.ts`](src/pwa.ts), [`src/sw-template.js`](src/sw-template.js), [`public/`](public/) |
| Troubleshooting clips | [`src/audio/clip.ts`](src/audio/clip.ts) |
| Reference sounds: player, source choice, plucked-string model | [`src/audio/reference-player.ts`](src/audio/reference-player.ts), [`src/audio/reference-source.ts`](src/audio/reference-source.ts), [`src/audio/pluck.ts`](src/audio/pluck.ts) |
| Building the recorded samples | [`scripts/make-samples.py`](scripts/make-samples.py), [`scripts/pick-samples.ts`](scripts/pick-samples.ts) |

### Adding an instrument

Add an entry to `INSTRUMENTS` in `src/instruments.ts`, listing the strings in physical order:

```ts
{
  id: 'cello',
  name: 'Cello',
  tunings: [{ id: 'standard', name: 'Standard (CGDA)', strings: ['C2', 'G2', 'D3', 'A3'] }],
},
```

It appears as a new tab, and its tunings appear in the tuning picker (the first one is the default). The detection range and filter settings are derived from the notes. To add a preset tuning to an existing instrument, add an entry to its `tunings` list.

## Benchmark

`npm run bench` runs the original MATLAB algorithm and the new one on synthetic tones and writes [`bench/results.md`](bench/results.md). The tones are pure sines, plucked strings, a weak fundamental, a noisy signal and a signal with DC offset, played on every string at several detunings.

| Share of readings within ±3 cents | Original | New |
|---|---|---|
| Guitar, plucked | 24% | 100% |
| Ukulele, plucked | 0% | 100% |
| Bass, plucked | 86% | 100% |
| Violin, plucked | 0% | 100% |
| Any instrument, 15 dB noise | 0–83% | 100% |

The new detector also rejects white noise, which the original reported as a pitch. It is also over 20× faster per frame (FFT-based), and its frames are shorter: 46 ms for guitar instead of 200 ms.

### Live responsiveness

`npm run bench:live` streams simulated microphone recordings through the live engine and writes [`bench/responsiveness.md`](bench/responsiveness.md). The recordings include a pick attack, a decay down to the room noise, and a peg being turned while re-plucking.

| While the string rings | Before | Now |
|---|---|---|
| Reading on screen, loud pluck | 90–92% | 98–100% |
| Reading on screen, quiet phone mic | 11–19% | 98–100% |
| Dropouts (reading vanished mid-note) | 6–27 per run | 0 |
| First reading after a pluck | 20–85 ms | 20–90 ms |

The engine learns the room's noise floor and follows a note down to it. Once a note is being tracked, noisier frames that agree with it continue it, while a new note needs a clean detection. Smoothing gets stronger as the note fades.

**Known limitation:** strong mains hum (50/60 Hz) on a wired setup interferes with bass E1 and A1, whose pitches sit close to it. Phones on battery are not affected.

### Real-recording sessions

[`uxtest/`](uxtest/) builds scripted tune-ups from real guitar, bass, violin and ukulele recordings. Each session has a detuned pluck, the peg turned while the string rings, an overshoot, re-plucks, room echo, background speech and bumps. They are played through the engine, or through the real app in a browser at phone CPU speed, and scored on what a player would notice. Results and fixes are in [`uxtest/RESULTS.md`](uxtest/RESULTS.md).

Three things that testing changed in the engine:
- A tracked note is measured on a copy of the signal filtered around its fundamental, so overtones don't pull the reading sharp.
- Octave jumps are rejected when the note being followed still fits the waveform.
- Only a real pluck, about as loud as recent ones, can switch to a different note while one is ringing.

## Development

Requires [Node.js](https://nodejs.org) 20 or newer.

```bash
npm install
npm run dev       # live-reloading app at http://localhost:5173
npm test          # unit tests
npm run bench     # algorithm benchmark → bench/results.md
npm run bench:live  # live responsiveness benchmark → bench/responsiveness.md
uxtest/run.sh       # real-recording tuning sessions → uxtest/RESULTS.md
npm run build     # one self-contained file: dist/index.html
```

## Deploying

`.github/workflows/deploy.yml` builds and tests the app and publishes it to GitHub Pages on every push to `main`. Turn it on once under **Settings → Pages → Build and deployment → Source: GitHub Actions**.
