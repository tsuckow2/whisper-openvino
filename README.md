# whisper-openvino

GitHub Actions build of [whisper.cpp](https://github.com/ggml-org/whisper.cpp)
`whisper-server` for Windows x64 with the encoder offloaded to
[OpenVINO](https://github.com/openvinotoolkit/openvino) — Intel CPUs and
Intel GPUs (integrated and discrete).

Built for the [llm-pool](https://github.com/codingwell/llm-pool) nodes:
a node whose dGPU is busy with chat models can run transcription on its
Intel iGPU instead of its CPU, by pointing the supervisor's
`whisper_server_bin` at this build.

## Run a build

Actions tab → **Whisper (OpenVINO, Windows)** → Run workflow.

- `whisper_ref`: whisper.cpp branch, tag, or SHA to build (default `master`).
- `smoke_model`: ggml model for the server smoke test (default `tiny.en`;
  empty skips the transcription check and only runs `--help`).

The workflow (manual trigger only) clones whisper.cpp, builds
`whisper-server.exe` and `whisper-cli.exe` with MSVC and
`-DWHISPER_OPENVINO=ON` against OpenVINO **2026.3.0** (the version
whisper.cpp's README recommends; the archive download is SHA256-verified),
checks the server really imports `openvino.dll`, stages a self-contained
bundle, smoke-tests the staged server by transcribing `jfk.wav`, and
uploads the bundle as the `whisper-server-openvino-windows` artifact
(90-day retention).

## Use the artifact

Copy the whole unzipped folder to the node and point the supervisor at it:

```json
"whisper_server_bin": "C:\\tools\\whisper-openvino\\whisper-server.exe"
```

The OpenVINO DLLs beside the exe are required; keep them together. A
Visual C++ Redistributable may be needed on a bare node.

Two things the binary alone does not do:

1. **The OpenVINO encoder only engages when a
   `ggml-<model>-encoder-openvino.xml/.bin` sits next to the ggml model.**
   Generate one per model with whisper.cpp's
   `models/convert-whisper-to-openvino.py` (Python 3.10, per the upstream
   README). Without it the server still transcribes, with the encoder on
   CPU.
2. **Pick the device with the entry's `extra_args`**, e.g.
   `["--ov-e-device", "GPU"]` — the default is CPU. The first run on a
   device is slow while OpenVINO compiles the IR to a device blob and
   caches it.

## Bumping OpenVINO

The archive filename carries a build number, so a version bump means
updating `OPENVINO_URL` and `OPENVINO_SHA256` together in
`.github/workflows/whisper-openvino.yml`. Both come from
<https://storage.openvinotoolkit.org/repositories/openvino/packages/> —
the `.sha256` file sits next to the `.zip` there.
