# Windows builds and verification

Builds for Akshat's two Windows laptops, the CI workflows and what each has verified. Exe hashes,
deployment state and test timings for the running copies belong in `../voice-workspace/setup.md`
and `../voice-workspace/performance.md`.

## CI workflows (GitHub Actions)

Full builds and test suites run here, not on the laptops. All workflows use a read-only token and
no secrets.

- `code-quality.yml` (push/PR on frontend paths, or manual): Bun 1.4.2, frozen lockfile, model
  language coverage, keyboard and history-delivery tests, ESLint, Prettier plus `cargo fmt`, and
  the production frontend build.
- `test.yml` (push/PR on `src-tauri/**`, or manual): `cargo test` on Ubuntu 24.04.
- `playwright.yml` (PR or manual): Playwright smoke tests against the dev server.
- `windows-arm64-build.yml` (manual only): the unsigned, CPU-only `aarch64-pc-windows-msvc` exe on
  the `windows-11-arm` runner, Rust 1.98.0, clang-cl + Ninja, `tauri build --no-bundle`. It checks
  that `--list-devices` shows a CPU and no GPU, and uploads a ready-to-run folder
  (`voice-workspace.exe`, the eight ARM64 VC++ runtime DLLs, `resources/` and a recursive
  `SHA256SUMS.txt`) as a 7-day artifact. There are no `ggml*`/`transcribe` DLLs in this build.
- `windows-x64-build.yml` (manual only): the unsigned `x86_64-pc-windows-msvc` exe with the Vulkan
  GPU backend for the x64 NVIDIA laptop, on `windows-2025`, Rust 1.98.0, Vulkan SDK 1.4.309.0 and
  `spirv-headers` from vcpkg. It checks that the Vulkan backend DLL was staged and that
  `--list-devices` runs (the runner has no GPU, so only its CPU shows), and uploads a
  ready-to-run folder (`voice-workspace.exe`, the transcription runtime DLLs, the VC++ runtime
  with OpenMP, `resources/` and a recursive `SHA256SUMS.txt`) as a 7-day artifact.
- `release.yml` (manual only): checks that `tauri.conf.json`, `Cargo.toml` and `package.json` agree
  on the version and that its tag is new, runs both builds above, verifies their checksums, zips
  each folder with a `VERSION.txt` as `voice-workspace-windows-arm64.zip` and
  `voice-workspace-windows-x64.zip`, and publishes GitHub Release `v<version>` with a
  `SHA256SUMS.txt` for the zips. Only its publish job gets `contents: write`. The fixed asset
  names keep `releases/latest/download/<zip>` stable for the install script.

Releases are unsigned and there is no installer; Handy's signing and installer workflows were
removed. Run a workflow on a branch with `gh workflow run <file> --ref <branch>`; pushing a
branch other than `main` starts nothing on its own.

## Current verified source

`main` holds Handy's final imported source (`73c09585`), this project's features and the
standalone makeover; its feature CI evidence is in `voice-workspace-features.md`.

Local recovery copies of the pre-import history are kept under `.git/local-recovery/` and
`refs/local-recovery/` on the laptop that did the import. They are local only; never push them.

## Snapdragon X laptop (ARM64)

No Rust, Bun or Visual Studio toolchain is installed; it installs the ARM64 zip from a GitHub
Release with Voice Workspace's `scripts/install-desktop.ps1`. Windows runs the unsigned exe without an application-control block
(Smart App Control off; nothing bypassed). The deployed copy and its rollback copies are recorded
in `../voice-workspace/setup.md`.

Earlier ARM64 builds, kept as rollback copies there: run 40252857590 (`12ee212`, first tested
build), run 40412889343 (`arm64-rewarm` `ff21566`, idle re-warm on the old engine) and run
40419914181 (`9cb3884`, adds early chunks on the old engine). The `arm64-rewarm` branch stays on
GitHub as their source.

## x64 laptop (NVIDIA)

Like the Snapdragon laptop, it no longer needs a local build: on that laptop, pull
`voice-workspace` and run `scripts/install-desktop.ps1`, which picks the x64 zip, renames the old
`runtime/handy/` to `runtime/desktop/`, keeps the old exe, DLLs and `resources/` in a rollback
folder and brings the `silero-vad-v6.2-F32.gguf` voice detector that Handy-era runtimes lack. Its
running copy is still an older Handy-era build until then. The first `windows-x64-build.yml` run
([run 40496231458](https://github.com/akshatksingh18/voice-workspace-desktop/actions/runs/40496231458),
commit `411d0a09`, about 26 min on a cold cache) passed: it staged `voice-workspace.exe`
(`d345a5dc…0c37`), `transcribe.dll`, `ggml.dll`, `ggml-base.dll`, `ggml-vulkan.dll`, ten per-CPU
`ggml-cpu-*` modules and the VC++ runtime with `vcomp140.dll`. On the GPU-less runner it listed
only the CPU; the NVIDIA device and live use need that laptop.

A local build remains possible with Rust 1.98, Visual Studio Build Tools 2026, Bun 1.4.2, Vulkan
SDK 1.4.309.0 and `spirv-headers:x64-windows` from vcpkg (installed on that laptop under
`%LOCALAPPDATA%\handy-build-deps`, an existing folder name). ONNX Runtime is no longer needed.
Windows Device Guard blocks Bun's generated `node_modules\.bin\tsc.exe` shim, so run the two
frontend stages directly, then the Tauri build without its frontend hook:

```powershell
$bun = Join-Path $env:APPDATA 'npm\bun.ps1'
& $bun install --frozen-lockfile
& $bun x tsc --pretty false
& $bun x vite build

$deps = Join-Path $env:LOCALAPPDATA 'handy-build-deps'
$env:VULKAN_SDK = Join-Path $deps 'VulkanSDK\1.4.309.0'
$env:CMAKE_PREFIX_PATH = (Join-Path $deps 'vcpkg\installed\x64-windows').Replace('\', '/')
$cmakeBin = 'C:\Program Files (x86)\Microsoft Visual Studio\18\BuildTools\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin'
$env:PATH = "$(Join-Path $env:APPDATA 'npm');$env:VULKAN_SDK\Bin;$cmakeBin;$env:PATH"
$config = '{"build":{"beforeBuildCommand":""}}'
bun x tauri build --no-bundle --ci --config $config
```

Use this Tauri build rather than bare `cargo build --release`: Tauri embeds the production
frontend, while a bare Cargo release can still point at the development URL. The last verified x64
build (Handy-era) stayed alive for 20 s after `--start-hidden --debug`, initialized its settings,
history and WebView2 profile, and passed NVIDIA/Vulkan inference; its accuracy and latency results
are in the Voice Workspace documents.

## macOS and Linux

Both laptops are Windows, so neither platform is a target. The code is kept because removing it
would be a large, risky edit for no gain. `test.yml` compiles the app and runs its Rust tests on
Linux on every backend change, so Linux at least builds; nothing is packaged or run there, and
macOS is not compiled at all.

## Local ASR diagnostic

`src-tauri/examples/asr_latency.rs` benchmarks batch or real-time-paced native streaming without
microphone, VAD, history or paste, and emits aggregate JSON only, never recognized text. With the
x64 environment above:

```powershell
Set-Location src-tauri   # from the repository root
cargo build --release --example asr_latency
```

Copy the exe from `target/release/examples/` beside the matching runtime DLLs as
`asr_latency.exe`, then use Voice Workspace's `scripts/benchmark-asr.ps1`. Do not mix native or
library versions. Pass an explicit device name and check the returned `bound_device`. Gold
fixtures and WAVs stay local. A synthetic TTS result does not prove accuracy for Akshat's voice or
that pasting works.
