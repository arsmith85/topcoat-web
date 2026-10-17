# topcoat-web: STATUS

**BLOCKED at rule 1. The engines are JUCE-free, but the code that turns them into TopCoat Free is JUCE code.**
Work stopped as the brief requires. Nothing has been built for the web yet. A ruling is needed (below).

Sources read: VirtualHouseMasterFile `78c6314` (main), its pins VirtualHouseDSP `5ee2200` and
VirtualHousePlatform `57cee25`.

## What TopCoat Free does to a file

Settings: 3 characters (Modern, Classic, Electronic) x 2 targets (Streaming -14 LUFS / -1 dBTP / 24-bit;
CD -10 LUFS / -0.5 dBTP / 16-bit at 44.1 kHz). No knobs.

The processing is not a fixed chain run once. It is a job (`engine/MasterJob.cpp:160` `MasterJob::run`):

1. Measure the whole source's integrated loudness (BS.1770) -> reference gain `-18 LUFS - measured`, put into the EQ's input gain (`MasterJob.cpp:40`).
2. Pass P: EQ -> Compressor -> Saturator over the whole file, latency trimmed, to a float temp file (`MasterJob.cpp:202`).
3. Measure pass P's loudness and true peak.
4. Up to 4 limiter passes at different drives, each measured, steered by `LoudnessStep` to hit the target inside the limiting cap (`MasterJob.cpp:227`).
5. Final limiter pass at the chosen drive -> TPDF dither -> WAV (`MasterJob.cpp:254`).

## What is JUCE-free (usable as-is)

| Part | Where |
|---|---|
| The four engines + settle/pre-roll | `engine/chain/MasterChain.h` |
| The 3 characters' numbers | `engine/chain/Characters.h` |
| Targets and constants | `engine/chain/Targets.h` |
| Loudness-step controller | `engine/chain/LoudnessStep.h` |
| Loudness + true-peak meters | VirtualHouseDSP `vhdsp/meter/Loudness.h`, `TruePeak.h` |
| TPDF dither | Platform `modules/vh_standalone/core/Dither.h` |

## What depends on JUCE (needed to sound like the app)

| What | Where (file:line) | JUCE use |
|---|---|---|
| The job itself: steps 1-5 above | VirtualHouseMasterFile `engine/MasterJob.cpp:160-290` | `juce::File`, `juce::AudioFormatManager`, readers/writers, `vh::standalone::render` |
| Reference-gain clamp | `engine/MasterJob.cpp:40` | `juce::jlimit` (trivial) |
| Drive below the limiter's range -> gain in front | `engine/MasterJob.cpp:93-97` | `juce::Decibels` (trivial) |
| Limiter measuring pass (latency alignment, meter feed) | `engine/MasterJob.cpp:102-140` | `juce::AudioBuffer`, file reader |
| Meter wrapper + whole-file measure | `engine/Measure.h:8` (include), `:59` `measureFile` | `juce_audio_formats`; the `Meter` class at `:26` is JUCE-free logic but lives in this JUCE header |
| Chain as render graphs | `engine/Graphs.h:10, :20` | `vh::standalone::RenderGraph` |
| Render loop: 512-sample blocks, silence padding, per-output latency trim, dither applied per frame | Platform `modules/vh_standalone/juce/Renderer.cpp:198-395` (trim `:350`, dither `:305`) | `juce::AudioBuffer`, `juce::AudioFormatWriter` |
| Dither seed (FNV-1a of lower-case output file name) | Platform `modules/vh_standalone/juce/Sources.cpp:37` | `juce::String` |

None of the JUCE use is DSP. It is file I/O and buffers around the job's logic. But that logic (steps 1-5,
the alignment, the dither order) is what makes the output TopCoat's, so a web build without it would not
sound the same as the app.

## Feasibility already checked (no work product, just evidence)

- emsdk installs here (emcc 6.0.12).
- A probe including `MasterChain.h`, `Characters.h`, `LoudnessStep.h`, the meters and `Dither.h` compiles with
  `em++ -O3` and runs under Node. It needs a larger stack than Emscripten's 64 KB default (`-sSTACK_SIZE`), since
  the chain object is large.
- Native g++ vs WASM on that probe (Electronic, 100 blocks of a sine): max difference about 1.3e-7. Not a parity
  result yet. Note: `vhdsp/core/pack2.h` uses SSE2 natively and plain doubles in WASM; the header says the lanes are
  bit-identical, which the parity test would have to confirm.

## Ruling needed (main-claude-cowork)

1. **Recommended: lift the job's logic into a JUCE-free file upstream.** In VirtualHouseMasterFile, move steps 1-5
   (measure, reference gain, pass P, the limiter passes, final pass with dither) into e.g.
   `engine/chain/MasterCore.h`, working on in-memory float buffers, with `MasterJob.cpp` reduced to file I/O around
   it. Move `Meter` out of `Measure.h`, and the 24-bit/16-bit quantise loop and `seedFor` out of the JUCE half of
   vh_standalone. The app and the web then compile the same code and cannot drift apart. That is a change to an
   engine repo (I can't push there), and it needs the app's own tests (MC-HASH etc.) to stay green.
2. Alternative: I re-implement `MasterJob::run` JUCE-free inside topcoat-web. Faster, but it is a second copy
   of the app's logic, which will drift the first time tuning changes `MasterJob.cpp`. I'd only do it with a
   native check against the real JUCE app build (fetch JUCE 9.0.2, run `mafree_render`) to prove they match.

## Other questions for the ruling (product, not code)

- **The 20 s window will not match the app's full-song master.** The app sets the reference gain and the limiter
  drive from loudness measured on the whole song. A demo that processes only 20 s measures only those 20 s, so its
  gain and drive differ from what the app does to the same song. Options: (a) measure on the whole decoded song,
  which means running pass P and the limiter passes on the whole song (render time scales with song length, so
  roughly 12x the 5 s phone budget for a 4-minute song); (b) measure the 20 s only, and say "the demo masters this
  section" rather than "this is what the app will do".
- **Which settings.** The app has 3 characters x 2 targets. The brief asks for 24-bit output; CD is 16-bit at
  44.1 kHz in the app. Proposed: the 3 characters on Streaming only.
- **MP3.** The app doesn't accept MP3 (WAV/AIFF/FLAC only). The demo can decode it in the browser, but then the
  demo accepts a file the app rejects.
- **Browser decoding.** `decodeAudioData` resamples to the context's rate. The demo has to read the file's rate
  from its header and decode at that rate, or the input already differs from what the app reads.

BLOCKED: awaiting ruling on rule 1 (R-TOPCOAT-WEB-1)
