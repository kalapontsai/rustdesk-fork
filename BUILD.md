# HelperDesk — Build Notes

Fork of [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk) with the
binary/app identity renamed to **HelperDesk**. Source-only changes so far;
no release binary has been produced yet.

## What was renamed

| Layer | Old | New |
|---|---|---|
| Master app-name constant (`APP_NAME` in `lazy_static`) | `RustDesk` | `HelperDesk` |
| Cargo package / lib / default-run | `rustdesk` / `librustdesk` | `helperdesk` / `libhelperdesk` |
| Cargo description | `RustDesk Remote Desktop` | `HelperDesk Remote Desktop` |
| Windows `BINARY_NAME` (CMake) | `rustdesk` | `helperdesk` |
| Windows VERSIONINFO (`.rc`) | ProductName `RustDesk`, InternalName `rustdesk`, OriginalFilename `rustdesk.exe`, FileDescription `RustDesk Remote Desktop`, Company `Purslane Tech Pte. Ltd.`, Copyright `Purslane ...` | `HelperDesk` / `helperdesk` / `helperdesk.exe` / `HelperDesk Remote Desktop` / `HelperDesk Project` / `HelperDesk Project` |
| Window-title fallback (`main.cpp`) | `L"RustDesk"` | `L"HelperDesk"` |
| Hardcoded UI text in `tabbar_widget.dart` | `Text("RustDesk", ...)` | `Text("HelperDesk", ...)` |
| 2FA issuer (`auth_2fa.rs`) | `"RustDesk"` | `"HelperDesk"` |
| `is_rustdesk()` / `is_custom_client()` comparisons | `"RustDesk"` | `"HelperDesk"` |
| IPC peer-identity check (`ipc/auth.rs`) | `OsStr::new("RustDesk"/"rustdesk")` | `OsStr::new("HelperDesk"/"helperdesk")` |
| Custom-client staging dir (`platform/windows.rs`) | `public\RustDesk\RustDeskCustomClientStaging` | `public\HelperDesk\RustDeskCustomClientStaging` |
| Update-channel owner/repo check | `owner == "rustdesk" && repo == "rustdesk"` | `owner == "helperdesk" && repo == "helperdesk"` |
| Portable crate metadata | `ProductName = "RustDesk"`, `IDENTIFIER = b"rustdesk"`, `APP_PREFIX = "rustdesk"` | `HelperDesk` / `b"helperdesk"` / `"helperdesk"` |
| Portable config writer (`generate.py`) | writes `"rustdesk"` twice | writes `"helperdesk"` twice |
| `libs/base/src/platform/mod.rs` platform string | `"RustDesk"` | `"HelperDesk"` |

### What was intentionally NOT changed

- `flutter_hbb` (Dart package name, 123 import sites) — internal to the
  Dart module graph, does not show in the binary or to the user.
- Android, macOS, Linux Flutter resources — out of scope for a Windows
  build; Windows-only was the request.
- The generic language-template substitution in `src/lang.rs` and
  `src/platform/macos.rs` — it already reads from `APP_NAME` at runtime,
  so it picks up the rename automatically.
- `res/icon.png` / `flutter/windows/runner/resources/app_icon.ico` — the
  default icon is kept; swap it and run
  `flutter pub run flutter_launcher_icons` to regenerate platform assets.

## Hash change

You do not need to bump versions to change the binary hash — rebuilding
already produces a different hash because the embedded
`APP_NAME = "HelperDesk"` string, the package metadata, the resource
VERSIONINFO block, and the rebuild timestamp all differ from upstream.
A small `Cargo.lock` bump of any transitive dep will shuffle things
further.

## Build prerequisites (matching upstream CI)

From `.github/workflows/flutter-build.yml` (`build-for-windows-flutter`,
matrix entry `x86_64-pc-windows-msvc` on `windows-2022`):

| Tool | Version |
|---|---|
| Rust | `1.75` (set via `rust-toolchain.toml` or `rustup override`) |
| Flutter | `3.24.5` (stable, x64) |
| LLVM / Clang | `15.0.6` |
| vcpkg | commit `9e593bb18ea69cc5095e012465dcd675a822ed0d` |
| CMake | `4.3.0` |
| MSVC | Visual Studio 2022 Build Tools (C++ workload, Windows 10/11 SDK) |

Disk: ~5 GB for toolchains + ~2 GB for build outputs. First clean build
takes 20–40 min; incremental rebuilds are minutes.

## Build steps (Windows host)

```powershell
# 1. Toolchain bootstrap
rustup default 1.75
# Install Flutter 3.24.5 to C:\flutter and add to PATH
# Install Visual Studio 2022 with C++ workload + Windows SDK
# Install vcpkg at the pinned commit and integrate with VS
git clone https://github.com/microsoft/vcpkg C:\vcpkg
cd C:\vcpkg && git checkout 9e593bb18ea69cc5095e012465dcd675a822ed0d
.\vcpkg.exe integrate install

# 2. Source
git clone <this-fork-url> helperdesk
cd helperdesk
git submodule update --init --recursive
git checkout <commit-with-rename>

# 3. Generate flutter_rust_bridge artifacts (one-time, requires the
#    `generate-bridge` workflow artifact, or run `flutter_rust_bridge_codegen`
#    locally with the matching Flutter 3.24.5)
# Easiest: just run `cargo build` once, which will trigger build.rs and
# the bridge codegen via flutter_rust_bridge = 1.80.1.

# 4. Build
cargo build --release --features hbb,flutter
# or, to drive via the official build script:
python build.py --flutter
```

The build produces:

- `target/release/librustdesk.dll` — Rust core (still named
  `librustdesk.dll` because the Windows CMake installs it as
  `librustdesk.dll`; rename that line in
  `flutter/windows/CMakeLists.txt` if you also want a renamed DLL).
- `flutter/build/windows/x64/runner/Release/helperdesk.exe` — the
  actual GUI executable.
- `flutter/build/windows/x64/runner/Release/data/flutter_assets/...`
  and bundled plugins.

## Build via GitHub Actions (no local toolchain)

1. Push this fork to a GitHub repo under your account.
2. In repo Settings → Secrets, add (or leave empty) `ANDROID_SIGNING_KEY`,
   `MACOS_P12_BASE64`, `SIGN_BASE_URL`, `SIGN_SECRET_KEY` — only the SIGN
   ones are used by the Windows job, and they are optional.
3. Trigger `.github/workflows/flutter-build.yml` via `workflow_dispatch`
   on branch `master` (or rename the trigger branches).
4. CI runs the same `build-for-windows-flutter` matrix as upstream. Pull
   the `rustdesk-unsigned-msi-template-x64` / `rustdesk-unsigned-exe`
   artifacts; for end-user distribution, sign the `.exe` with an EV
   code-signing cert (otherwise SmartScreen will red-screen first run).

## Caveats / known gotchas

- `flutter_hbb` Dart package name is unchanged. If you do change it
  later, every `package:flutter_hbb/...` import must be updated (123
  files). `find . -path ./node_modules -prune -o -name '*.dart' -print
  | xargs sed -i 's|package:flutter_hbb/|package:flutter_helperdesk/|g'`
  will get you most of the way; the `pubspec.yaml` `name:` line is the
  one to flip first.
- `librustdesk.dll` is still loaded by name from `main.cpp`:
  `LoadLibraryA("librustdesk.dll")`. The DLL is renamed in the Rust
  CMake install step (`RENAME librustdesk.dll`); if you also want the
  DLL renamed, change the `RENAME` clause **and** the `LoadLibraryA`
  call to match.
- The Windows service binary (`service.exe`) is built from
  `src/service.rs` and uses `default-run` for the GUI. Both share the
  `APP_NAME` constant, so they will both identify as `HelperDesk`.
- `flutter/assets/` does not embed any user-visible "RustDesk" string,
  but `flutter_icons` regeneration is needed if you swap
  `res/icon.png`.
- Upstream's MSI template (`flutter/windows/runner/...`) hardcodes the
  binary as `rustdesk.msi`; if you publish an MSI, also patch
  `flutter/windows/runner/CMakeLists.txt` install rules and the
  `.wxs`/`.wxi` if any.
