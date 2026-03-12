# Native Build Process

This document describes the full native build pipeline for `com.unity.webrtc`.

There are two separate layers in the native build:

1. Build the platform-specific `libwebrtc` package from WebRTC source.
2. Build the Unity native wrapper plugin against that packaged `libwebrtc`.

The repository normally consumes prebuilt `libwebrtc` archives published in Unity release artifacts. You only need to run the `build_libwebrtc_*` scripts when you want to rebuild or modify the underlying WebRTC libraries themselves.

## Repository layout

- `BuildScripts~`: entry-point scripts for building `libwebrtc`, the Unity plugin wrapper, and native test runners.
- `Plugin~`: the CMake project for the native wrapper plugin.
- `Plugin~/webrtc`: unpacked `libwebrtc` headers and libraries consumed by CMake.
- `Runtime/Plugins`: final native artifacts loaded by Unity.

## Build pipeline overview

The normal wrapper build flow is:

1. Run the platform `build_plugin_*` script from the repository root.
2. The script downloads a prebuilt `webrtc-<platform>.zip` archive from the Unity release page.
3. The archive is unpacked into `Plugin~/webrtc`.
4. CMake discovers those headers and libraries through `Plugin~/cmake/FindWebRTC.cmake`.
5. The `Plugin~/WebRTCPlugin` project builds:
   - `WebRTCLib`: the static wrapper implementation.
   - `WebRTCPlugin`: the final dynamic library or framework exported to Unity as `webrtc`.
6. The resulting artifact is copied or emitted into `Runtime/Plugins/<platform>`.

The source `libwebrtc` build flow is:

1. Run the platform `build_libwebrtc_*` script from the repository root.
2. The script clones `depot_tools` if needed.
3. It fetches the upstream WebRTC checkout, syncs dependencies, and checks out branch `5845`.
4. Unity-specific patches from `BuildScripts~/patches` are applied.
5. `gn gen` and `ninja` build the platform WebRTC static libraries.
6. Headers, libraries, and license files are packaged into `webrtc-<platform>.zip` under `artifacts`.

## Build from source or use release artifacts

Use `build_plugin_*` when:

- you are changing code under `Plugin~/WebRTCPlugin`
- you want to rebuild the Unity native wrapper only
- you are fine using the published `libwebrtc` binaries

Use `build_libwebrtc_*` first when:

- you are changing upstream WebRTC behavior
- you need to modify or validate Unity's WebRTC patch set
- you need a new platform archive to consume from `build_plugin_*`

## CMake wiring

The wrapper build is driven from `Plugin~/CMakeLists.txt`.

- `find_package(WebRTC REQUIRED)` resolves the unpacked `Plugin~/webrtc` tree.
- `add_subdirectory(WebRTCPlugin)` builds the native wrapper targets.
- `Plugin~/WebRTCPlugin/CMakeLists.txt` selects platform-specific link flags, system libraries, and output directories.

`Plugin~/cmake/FindWebRTC.cmake` expects this layout under `Plugin~/webrtc`:

- `include/...`
- `lib/x64/...` for Windows and Linux
- `lib/<android-arch>/...` for Android
- `lib/libwebrtc.a` and `lib/libwebrtcd.a` for Apple universal binaries
- `lib/libwebrtc.aar` for Android packaging

## Building the Unity wrapper plugin

Run these scripts from the repository root.

### Windows

Command:

```bat
BuildScripts~\build_plugin_win.cmd [debug]
```

What it does:

- downloads `webrtc-win.zip`
- unpacks it into `Plugin~/webrtc`
- configures CMake with preset `x64-windows-msvc`
- builds target `WebRTCPlugin` with preset `release-windows-msvc`

Output:

- `Runtime/Plugins/x86_64/webrtc.dll`

Notes:

- the wrapper links Vulkan, CUDA, and NVIDIA codec support on desktop Windows
- the build script currently uses the MSVC preset even though the older plugin README still mentions Clang

### Linux

Command:

```bash
./BuildScripts~/build_plugin_linux.sh [debug]
```

What it does:

- downloads `webrtc-linux.zip`
- unpacks it into `Plugin~/webrtc`
- configures CMake with preset `x86_64-linux`
- builds target `WebRTCPlugin` with preset `release-linux`
- strips unused symbols from the resulting shared object

Output:

- `Runtime/Plugins/x86_64/libwebrtc.so`

Notes:

- Linux uses Clang 11 and statically links `libstdc++` for compatibility with older distributions
- desktop Linux also enables Vulkan and NVIDIA codec support

### macOS

Command:

```bash
./BuildScripts~/build_plugin_mac.sh [debug]
```

What it does:

- downloads `webrtc-mac.zip`
- unpacks it into `Plugin~/webrtc`
- configures CMake with preset `macos`
- builds target `WebRTCPlugin` with preset `release-macos`

Output:

- `Runtime/Plugins/macOS/libwebrtc.dylib`

Notes:

- the macOS wrapper links Apple frameworks through `FindFramework.cmake`
- the consumed WebRTC archive contains a universal Apple static library

### iOS

Command:

```bash
./BuildScripts~/build_plugin_ios.sh [debug]
```

What it does:

- downloads `webrtc-ios.zip`
- unpacks it into `Plugin~/webrtc`
- configures CMake for `CMAKE_SYSTEM_NAME=iOS` using Xcode
- builds the static wrapper library for simulator and device
- archives the `WebRTCPlugin` framework for simulator and device separately
- copies the device archive product into the Unity package

Output:

- `Runtime/Plugins/iOS/webrtc.framework`

Notes:

- the CMake target is marked as a framework on iOS
- the script currently copies the device framework only
- there is commented-out `lipo` logic in the script for combining simulator and device binaries, but it is intentionally disabled

### Android

Command:

```bash
./BuildScripts~/build_plugin_android.sh [debug]
```

What it does:

- downloads `webrtc-android.zip`
- unpacks it into `Plugin~/webrtc`
- copies `lib/libwebrtc.aar` into `Runtime/Plugins/Android`
- configures and builds `WebRTCPlugin` twice, once for `arm64-v8a` and once for `x86_64`
- takes the resulting `libwebrtc.so` for each ABI and injects it into the AAR under `jni/<abi>`

Output:

- `Runtime/Plugins/Android/libwebrtc.aar`

Notes:

- the Android wrapper is not distributed as a standalone `.so`; it is merged into the AAR
- debug builds optionally add the Vulkan validation layer shared library to the AAR
- the build requires `ANDROID_NDK` to be set

## Rebuilding libwebrtc from source

These scripts build the packaged `webrtc-<platform>.zip` inputs used by the wrapper scripts.

### Common behavior

All `build_libwebrtc_*` scripts:

- clone `depot_tools` if needed
- fetch the correct WebRTC checkout for the target platform
- check out WebRTC branch `5845`
- apply patches from `BuildScripts~/patches`
- build debug and release variants where applicable
- collect `include`, `lib`, and `LICENSE.md` into `artifacts`
- create a platform zip archive for later consumption

### Windows

Command:

```bat
BuildScripts~\build_libwebrtc_win.cmd
```

Output archive:

- `artifacts/webrtc-win.zip`

Packaged libraries:

- `artifacts/lib/x64/webrtc.lib`
- `artifacts/lib/x64/webrtcd.lib`

### Linux

Command:

```bash
./BuildScripts~/build_libwebrtc_linux.sh
```

Output archive:

- `artifacts/webrtc-linux.zip`

Packaged libraries:

- `artifacts/lib/x64/libwebrtc.a`
- `artifacts/lib/x64/libwebrtcd.a`

### macOS

Command:

```bash
./BuildScripts~/build_libwebrtc_macos.sh
```

Output archive:

- `artifacts/webrtc-mac.zip`

Packaged libraries:

- `artifacts/lib/libwebrtc.a`
- `artifacts/lib/libwebrtcd.a`

Notes:

- macOS builds both `x64` and `arm64` and merges them with `lipo`

### iOS

Command:

```bash
./BuildScripts~/build_libwebrtc_ios.sh
```

Output archive:

- `artifacts/webrtc-ios.zip`

Packaged libraries:

- `artifacts/lib/libwebrtc.a`
- `artifacts/lib/libwebrtcd.a`

Notes:

- iOS builds device `arm64` and simulator `x64` variants and merges them with `lipo`

### Android

Command:

```bash
./BuildScripts~/build_libwebrtc_android.sh
```

Output archive:

- `artifacts/webrtc-android.zip`

Packaged libraries:

- `artifacts/lib/arm64/libwebrtc.a`
- `artifacts/lib/arm64/libwebrtcd.a`
- `artifacts/lib/x64/libwebrtc.a`
- `artifacts/lib/x64/libwebrtcd.a`
- `artifacts/lib/libwebrtc.aar`
- `artifacts/lib/libwebrtc-debug.aar`

Notes:

- Android produces both static libraries and an AAR
- the AAR is built through `tools_webrtc/android/build_aar.py`

## Typical workflows

### Change wrapper code only

1. Edit code under `Plugin~/WebRTCPlugin`.
2. Run the matching `build_plugin_*` script.
3. Verify the new artifact under `Runtime/Plugins/<platform>`.

### Change underlying WebRTC code

1. Edit the WebRTC checkout patches under `BuildScripts~/patches` or update the source-build script.
2. Run the matching `build_libwebrtc_*` script.
3. Replace or publish the generated `webrtc-<platform>.zip`.
4. Run the matching `build_plugin_*` script so the wrapper is rebuilt against the new package.

## Known mismatches and caveats

- `Plugin~/README.md` still contains some older environment notes, including Windows guidance that mentions Clang and a Visual Studio 2019 workaround. The current checked-in wrapper script uses the Visual Studio 2022 MSVC preset.
- The iOS wrapper script archives both simulator and device builds, but only the device framework is copied into `Runtime/Plugins/iOS`.
- Android requires `ANDROID_NDK` in the environment; the CMake preset file also contains an older `CMAKE_ANDROID_NDK` default pointing at `ANDROID_SDK`, but the wrapper build script passes `ANDROID_NDK` explicitly.

## Related files

- `Plugin~/README.md`: environment setup and historical wrapper build notes
- `Plugin~/CMakeLists.txt`: native CMake entry point
- `Plugin~/WebRTCPlugin/CMakeLists.txt`: wrapper target definitions and platform-specific output rules
- `Plugin~/cmake/FindWebRTC.cmake`: locates unpacked `Plugin~/webrtc` headers and libraries