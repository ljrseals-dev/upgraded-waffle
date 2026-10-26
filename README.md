Copyright © 2026 Lewis James Seals Jr. All rights reserved.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT.
# ElevenX — Drop-in TTS + HarmoniaCodec Promotion

**Branch target:** `feature/dropin-tts-harmonia`  
**Source package:** `dropin-tts-harmonia.zip`  
**Status:** Ready to apply. XTTS-v2 remains the default synthesis path.

## What this is

A finished promotion of the numpy-only drop-in TTS engine into the ElevenX
layout. It adds:

1. A zero-dependency formant TTS (`baseline`) usable offline and in CI.
2. A reserved **HarmoniaCodec** slot that falls back safely until trained
   weights and a quality report exist.
3. Docs, tests, and a thin bridge so the path can be called without touching
   the Coqui XTTS pipeline.

This does **not** replace XTTS, enable HarmoniaCodec in production, or make
any quality claims about the codec.

## Layout (apply relative to ElevenX repo root)

```
src/elevenx/tts/                 # vendored drop-in package
  __init__.py
  cli.py
  dsp.py
  engine.py
  frontend.py
  codec/
    __init__.py
    interface.py                 # frozen BaseCodec / CodecLatent / CodecConfig
    registry.py                  # string-name registry + autoload
    baseline.py                  # Klatt-style formant synthesizer
    harmonia.py                  # reserved slot (NotImplemented → fallback)
src/elevenx/tts_bridge.py        # optional ElevenX-style entry point
tests/test_tts_dropin.py         # model-free tests
docs/
  dropin-tts-harmonia-promotion.md
  harmoniacodec-integration.md   # (existing boundary — keep)
  README-dropin-section.md       # snippet to merge into main README
  config-codec-snippet.yaml      # optional config.yaml extension
```

## Architecture

```
text → frontend (phonemes + prosody)
     → codec registry (baseline | harmonia)
     → encode → decode
     → NoiseBed → MultibandCompressor → ProDither
     → mono float32 / WAV
```

Codec selection is string-based. `engine.py` never imports `codec.harmonia`
directly. When Harmonia is selected but not installed, the engine emits one
`RuntimeWarning` and regenerates with the baseline codec. Latents are never
decoded by the wrong decoder.

## Quick use (after apply)

```python
from elevenx.tts import TTSEngine, available_codecs

print(available_codecs())  # ['baseline', 'harmonia']

engine = TTSEngine(codec_name="baseline", sample_rate=22050)
audio = engine.synth("Welcome to ElevenX.")
# audio: float32 mono, peak-normalized ≈ 0.95
```

```bash
python -m elevenx.tts.cli "hello world" out.wav
python -m elevenx.tts.cli "hello world" out.wav harmonia   # falls back with warning
```

Bridge (optional):

```python
from elevenx.tts_bridge import synthesize_dropin
result = synthesize_dropin("Welcome to ElevenX.", codec="baseline")
print(result.sample_rate, result.duration_seconds, result.codec)
```

## Apply steps

From the ElevenX repository root:

```bash
unzip elevenx-dropin-tts-harmonia-promotion.zip
# or: copy the tree over existing src/elevenx/ and docs/

git checkout -b feature/dropin-tts-harmonia
git add src/elevenx/tts src/elevenx/tts_bridge.py tests/test_tts_dropin.py docs/
# Optionally paste docs/README-dropin-section.md into README.md
# Optionally extend config.yaml from docs/config-codec-snippet.yaml

python -m pip install numpy pytest   # if not already present
PYTHONPATH=src python -m pytest tests/test_tts_dropin.py -q

git commit -m "Promote drop-in TTS engine with HarmoniaCodec slot (fallback-safe)"
git push -u origin feature/dropin-tts-harmonia
```

## Completing real HarmoniaCodec application

Still required before any production enablement (unchanged policy):

1. Implement `load` / `encode` / `decode` in `src/elevenx/tts/codec/harmonia.py`
   against a trained checkpoint from [HarmoniaCodec](https://github.com/ljrseals-dev/HarmoniaCodec).
2. Produce a `CodecQualityReport` (`weights_sha256`, non-empty `benchmark_id`,
   `passed=True`) and match it to `expected_weights_sha256` in config.
3. Confirm `check_harmonia_readiness(...).ready` is `True`.
4. Only then set `codec.enabled: true` in `config.yaml`.
5. Keep XTTS as default/fallback until end-to-end quality gates pass.

See `docs/dropin-tts-harmonia-promotion.md` and the existing
`docs/harmoniacodec-integration.md`.

## Design constraints (do not regress)

- XTTS-v2 remains the default for `ElevenXEngine.synthesize` and
  `POST /v1/synthesize`.
- Synthesis stays local-first; no telemetry or remote model calls.
- Concurrent synthesis still serializes (HTTP 503 + Retry-After).
- Consent and reference-audio validation remain required for voice cloning.
- No trained Harmonia weights ship in this promotion; random initialization
  is not treated as a trained codec.

## License / provenance

Drop-in sources originate from the `dropin-tts-harmonia` package (same author
as ElevenX / HarmoniaCodec). Apply under the same repository license terms
as the rest of ElevenX.
