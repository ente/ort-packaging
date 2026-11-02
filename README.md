# Custom ONNX Runtime Packaging

Builds pinned, self-contained ONNX Runtime runtime packages for Android, iOS, Linux, macOS, and Windows without consuming Microsoft-provided ONNX Runtime binaries. Platform policy lives in [`build.env`](build.env), and pinned build tools in [`config/toolchains.json`](config/toolchains.json). Each release build resolves its requested upstream tag to an immutable commit.

This repository's release tags describe a **custom packaging** of upstream ONNX Runtime, not an upstream Microsoft release. Stable package revisions are named `ort-<upstream-version>-r<revision>`. See [Releases](https://github.com/ente/ort-packaging/releases) for available versions and assets.

## Release process

A routine ONNX Runtime release requires no repository changes. Do not edit `build.env` to select an ONNX Runtime version; change it only when the shared build policy or toolchain must change.

Make sure the packaging code you want to release is on `main`, then follow these three steps. The workflows enforce every other release requirement and stop if one fails.

### 1. Build a release candidate

Open [Build release candidate](https://github.com/ente/ort-packaging/actions/workflows/build.yml), select **Run workflow**, and use:

- **Branch:** `main`
- **ORT version:** the stable upstream version without a `v` prefix, such as `1.28.1`
- **Package revision:** `1` for the first package of that ORT version; use the next integer only to correct the packaging. Revisions must be sequential.

Leave the two preselected choices, `release` and `all`, unchanged.

The equivalent CLI command is:

```sh
gh workflow run build.yml \
  --repo ente/ort-packaging \
  --ref main \
  -f ort_version=1.28.1 \
  -f package_revision=1
```

The workflow verifies the upstream release, resolves its tag to a commit, applies the repository patches, and builds and tests every package. The candidate is ready when the run succeeds. If a job fails for a temporary reason, such as a network error, select **Re-run failed jobs** on that run.

### 2. Test on physical devices

Test Android WebGPU on representative supported physical devices and validate the iOS device slice and vendor-GPU behavior. Hosted CI validates package structure and runtime inference, but it cannot replace these device checks.

### 3. Publish the candidate

The successful build's summary page shows its run ID. Open [Publish release](https://github.com/ente/ort-packaging/actions/workflows/publish.yml), select **Run workflow** on `main`, and enter that ID.

The equivalent CLI command is:

```sh
gh workflow run publish.yml \
  --repo ente/ort-packaging \
  --ref main \
  -f build_run_id=123456789
```

The release is done when this run succeeds. Candidates expire after 14 days; if yours has expired, run a new build instead.

Never replace or retag an existing release. Create the next package revision instead.

## Trial builds

To check an ORT upgrade or packaging change before merging, run [Build release candidate](https://github.com/ente/ort-packaging/actions/workflows/build.yml) on your branch, choosing `trial` instead of `release`, and the affected platform or `all`. Trials use the release build scripts and may reuse an existing package revision, but they never produce a publishable candidate.

```sh
gh workflow run build.yml \
  --repo ente/ort-packaging \
  --ref your-branch \
  -f mode=trial \
  -f platform=linux \
  -f ort_version=1.28.1 \
  -f package_revision=1
```

## Packages

| Platform | Architectures | Asset family | Contents |
| --- | --- | --- | --- |
| Android | arm64-v8a, armeabi-v7a, x86_64 | `onnxruntime-webgpu-android-*` | API 24+ ABI-specific and universal AARs with ORT, Java/JNI, built-in WebGPU, XNNPACK, and CPU fallback |
| iOS | ARM64 device and Apple Silicon Simulator | `onnxruntime-coreml-ios-*` | Static-library XCFramework with CoreML and CPU; deployment target 15.1 |
| Linux | x64, ARM64 | `onnxruntime-webgpu-linux-*` | GLIBC 2.28-compatible ORT runtime plus a shared WebGPU plugin |
| macOS | Intel x64, Apple Silicon ARM64 | `onnxruntime-coreml-macos-*` | CoreML-enabled ORT runtime with CPU fallback; deployment target 13.3; no WebGPU plugin |
| Windows | x64, ARM64 | `onnxruntime-webgpu-windows-*` | ORT runtime, WebGPU plugin, required DXC DLLs, and DXC licences |

The provider in an asset name is intentional: Android/Linux/Windows support WebGPU, while Apple packages intentionally use CoreML rather than WebGPU. Do not mix a packaged runtime, plugin, or headers with another ONNX Runtime build.

Telemetry is compiled out of every package. Every build explicitly sets `onnxruntime_USE_TELEMETRY=OFF`, uses ONNX Runtime's `--no_telemetry` option when that release supports it, and rejects generated CMake configurations unless telemetry is disabled.

## Consumer requirements

- Android: the AAR has a package minimum of API 24. WebGPU still depends on the physical device's Android/Vulkan GPU support; CPU and XNNPACK remain fallbacks.
- iOS: link `Foundation`, weak-link `CoreML`, and link `c++`. CPU and standard CoreML usage support iOS 15.1. CoreML float16 inputs and ANE-only execution require iOS 16+, while CoreML specialization strategies require iOS 18+ and fail with a clear runtime error on older systems.
- macOS: link `Foundation`, weak-link `CoreML`, and target macOS 13.3+. CoreML is built into the core runtime; there is no WebGPU plugin to register.
- Linux: install a compatible Vulkan loader and driver for WebGPU. The package ABI supports Ubuntu 20.04 and newer (GLIBC 2.28 maximum, GLIBCXX 3.4.28 maximum), but GPU-driver correctness and performance remain environment-specific.
- Windows: install the Microsoft Visual C++ 2015–2022 runtime, as required by Microsoft's own ONNX Runtime binaries. Keep the DXC DLLs that ship in the archive beside the ORT runtime and WebGPU plugin.

iOS frameworks include their public headers. Linux, macOS, and Windows archives are runtime packages: C/C++ consumers should obtain the matching public headers from the pinned upstream source revision (or vendor them with their own build) rather than mixing headers from another ORT version.

For Linux and Windows, register the shared WebGPU plugin and select an EP device through ORT's V2 device API or automatic device selection. The legacy built-in WebGPU registration API does not load plugin EPs.

## Repository layout

- `build.env`: shared toolchain, deployment-target, ABI, and telemetry policy
- `config/`: platform build configurations and checksum-pinned CMake/vcpkg downloads
- `scripts/`: workflow implementation helpers for platform builds, packaging, and packaged-runtime smoke tests
- `patches/`: reviewable upstream compatibility patches; see [`PATCHES.md`](PATCHES.md)
- `tests/`: portable packaged-runtime inference smoke tests and release-script regression tests
- [`RELEASE-CHECKLIST.md`](RELEASE-CHECKLIST.md): manual checks required before publishing a stable release
- `.github/workflows/build.yml`: version-driven native build and release-candidate assembly
- `.github/workflows/publish.yml`: immutable publication of one successful build run

The build and smoke-test helpers under `scripts/` are workflow implementation details, not supported standalone release commands. The release workflow supplies their resolved metadata and writes to `.build/` and `.dist/`; local fallback directories `build/` and `dist/` are also ignored by Git.

## Build and validation design

The build workflow builds every target on its native GitHub-hosted runner. Linux compilation runs in checksum-pinned PyPA `manylinux_2_28` job containers on native x64 and ARM64 hosts, decoupling the package ABI from the GitHub runner image. Android ABIs are built independently and merged into a universal AAR. iOS is distributed for ARM64 devices and Apple Silicon Simulator hosts; Intel Simulator hosts are not supported. Its XCFramework contains ordinary `libonnxruntime.a` slices and public headers, so Swift Package Manager and consumers that link the archive directly use the same single copy of the machine code.

## Verification and provenance

Every binary asset includes a `.sha256` checksum and `.manifest.env` file. Manifests record the upstream ref and commit, packaging commit, telemetry policy, package channel/revision/label, target, provider topology, and build configuration. The release also includes `SHA256SUMS` and `build-provenance.json`, for 35 assets in total: 11 packages, their 11 checksum sidecars, their 11 manifests, and those two files.

A release build runs only for all platforms on `main`, and only after **Validate** succeeded on a push of that commit. Publication accepts only a successful `build.yml` run from this repository's `main` branch. It revalidates the candidate provenance, checksums, telemetry policy, upstream commit, package revision, and target tag before creating an immutable release from the exact packaging commit that produced the candidate. It then confirms that the published release is immutable, has all 35 assets, and has its tag on that commit.

Custom-built binaries are stripped of or packaged without debug-symbol data. Linux and Android builds additionally reject debug/symbol-table sections and executable stacks. Archives contain the ONNX Runtime licence and third-party notices; Android AARs store them under `META-INF/`. Windows archives include the licence files shipped with the pinned DXC runtime.

Always pin a release tag and verify its checksum sidecar downstream.

## Validation scope

CI verifies package structure, native architectures, required exports, Android JNI contents, checksums, provenance metadata, and packaged runtime loading. It requires CPU and WebGPU inference on Linux x64/ARM64 using Mesa's software Vulkan driver. Windows requires CPU inference and WebGPU plugin registration; D3D12 inference is skipped when the runner has no GPU device, as on both Windows runners for 1.31.0-r1, and no manual release check replaces it. It requires CPU and CoreML inference without CPU fallback on macOS x64/ARM64 and in an ARM64 iOS Simulator, plus packaged CPU inference in an Android x86_64 emulator. Numeric checks reject NaN and infinity.

Hosted Android emulators cannot exercise WebGPU without a host GPU. Android WebGPU and Android ARM device ABIs require representative physical-device validation before a stable release; iOS device-slice and vendor-GPU performance validation are also required before publishing. Smoke tests prove runtime loading and basic provider inference, not application-specific operator coverage or performance.

ONNX Runtime and bundled third-party components retain their upstream licences. Runtime archives include the applicable licence and notices.
