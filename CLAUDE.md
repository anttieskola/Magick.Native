# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Magick.Native is the native C shim that **Magick.NET** (separate repo, `dlemstra/Magick.NET`, usually checked out next to this one at `Z:\Magick.NET`) P/Invokes into. It is a thin layer of exported C functions over ImageMagick 7 (MagickCore + MagickWand), statically linked together with ImageMagick and all its delegate libraries into a single shared library per platform/architecture/quantum:

- `Magick.Native-<Quantum>[-OpenMP]-<arch>.dll` on Windows (`.dll.so` on Linux, `.dll.dylib` on macOS; the `.dll` suffix is kept on every platform)
- `magick.js` + `magick.wasm` for the browser (Q8 only), published to npm as `@dlemstra/magick-native`

There are **no tests in this repo**. Correctness is exercised by the Magick.NET test suite after Magick.NET bumps its pinned `src/Magick.Native/Magick.Native.version` to a new release of this repo. CI only verifies that the produced binaries load (`ldd`/`ld` checks against a matrix of Docker images, see `build/shared/verify.native.sh`).

## Building

ImageMagick must be built first, then this library is built against it. Both halves are driven by `.github/workflows/main.yml`; the scripts in `build/` are what CI runs and are meant to be executed the same way locally. The ImageMagick commit is pinned in `src/ImageMagick/ImageMagick.commit` (the nightly scheduled CI run overwrites it with `latest`). Prebuilt delegate libraries are downloaded as `<platform>-<arch>-static.zip` artifacts from the ImageMagick repo rather than built locally.

### Windows (Visual Studio 2026, run from a plain `cmd`; scripts call `VsDevCmd.bat` themselves)

```cmd
cd src\ImageMagick
checkout-release.cmd x64 noOpenMP          :: clones ImageMagick + downloads prebuilt dependencies via Git Bash
cd ..\..\build\windows
build.cmd Release Q16 x64 hdri noOpenMP    :: args: Debug|Release, Q8|Q16, x86|x64|arm64, hdri|noHdri, openMP|noOpenMP
copy-native.cmd ..\..\artifacts ReleaseQ16-HDRI x64
copy-resources.cmd ..\..\resources-artifacts
```

`build.cmd` runs ImageMagick's `Configure.Release.x64.exe` wizard non-interactively, builds `IM7.Static.<arch>.sln`, then builds `Magick.Native.sln` with configuration `<Debug|Release><Quantum>[-HDRI][-OpenMP]` and platform `x86|x64|arm64`. Output lands in `src/Magick.Native/bin/<Configuration>/<arch>/`. In `Magick.Native.vcxproj` the configuration name alone selects the quantum (`QuantumType` property) and the output file name; `x86` and OpenMP cannot be combined. To rebuild only this project once ImageMagick is built, open `Magick.Native.sln` in VS or run:

```cmd
msbuild Magick.Native.sln /p:Configuration=ReleaseQ16-HDRI,Platform=x64
```

For a debuggable build use a `Debug*` configuration. ASAN was deliberately removed from Debug because it breaks debugging from .NET.

The scripts invoke executables in the current directory by bare name (`build.cmd`, `Configure.Release.x64.exe`, `vswhere.exe`). The Claude Code shell sets `NoDefaultCurrentDirectoryInExePath=1`, which makes those calls fail with "is not recognized". When running them from here, clear it first: `cmd /c "set NoDefaultCurrentDirectoryInExePath=&& cd /d Z:\Magick.Native\build\windows && build.cmd ..."`. `Building.md` has a full step-by-step walkthrough with the expected output of each step.

`checkout-development.cmd` clones the Dependencies repo instead of downloading prebuilt artifacts, for working on ImageMagick itself.

### Linux / Linux-musl / macOS / WASM

Each target has a folder `build/<platform>-<arch>/` with `install.build-tools.sh` (apt/apk/brew/pip setup for a fresh container) and `settings.sh` (compiler flags, `CMAKE_COMMAND`, `EXTENSION`, `PLATFORM`; `DEBUG_BUILD=yes` in the environment switches to `-g3 -O0`). The shared scripts source that `settings.sh` based on their first two arguments:

```bash
# inside the matching container (ubuntu:20.04, alpine:3.21, macos-14, emscripten/emsdk)
cd build/linux-x64 && ./install.build-tools.sh
cd src/ImageMagick && ./checkout.sh --no-configure --dependencies-artifact linux-x64-static.zip
cd ImageMagick && ../../../build/shared/build.imagemagick.sh linux x64 Q16-HDRI OpenMP   # installs to /tmp/ImageMagick
cd ../../Magick.Native && ../../build/shared/build.native.sh linux x64 Q16-HDRI OpenMP    # cmake+make into src/Magick.Native/<Quantum>/
../../build/shared/copy.native.sh ../../artifacts linux x64 Q16-HDRI OpenMP
```

Platform names are `linux`, `linux-musl`, `macos`, `wasm`; quantum is `Q8|Q16|Q16-HDRI`; the fourth argument is literally `OpenMP` or anything else. WASM uses `build/wasm-<x86|x64>/copy.native.sh` instead of the shared one. Linux arm64 is cross-compiled on x64 with the `aarch64-linux-gnu` toolchain (`cross-compilation.cmake`), not built natively. macOS builds run natively with Homebrew tools, and `build/macos-arm64/install.build-tools.sh` deliberately uninstalls any brew-provided image libraries so only the downloaded static dependencies get linked. `.devcontainer/<target>/devcontainer.json` wraps all of this in a Docker image whose `post-create-command.sh` builds the library, which is the easiest way to get a non-Windows build on a Windows machine. CMake hard-codes ImageMagick at `/tmp/ImageMagick` and dependencies at `/tmp/dependencies` (see `src/Magick.Native/CMakeLists.txt`).

### WASM npm package

`src/wasm-file-creator` (TypeScript) scans every `.c` file for lines starting with `MAGICK_NATIVE_EXPORT` and generates `magick.d.ts` typings plus `magick.constants.ts` (embedding `Resources/xml/log.xml` and `policy.xml`). `publish/wasm/publish.sh --version X` then produces the npm tarball. Because the parser is line-based, **every exported function signature must stay on a single line** starting with the macro.

## Code layout and conventions

All sources live under `src/Magick.Native/`, grouped by the managed type they back (`MagickImage.c`, `MagickImageCollection.c`, `Settings/`, `Types/`, `Statistics/`, `Pixels/`, `Drawables/DrawingWand.c`, `OpenCL/`, ...). Every `.c` has a matching `.h` that declares the exports inside `extern "C"` guards. The sources are compiled **as C++** on every platform (`CompileAsCpp` in the vcxproj, `LANGUAGE CXX` in CMake), so casts must be explicit. New `.c` files must be added to `Magick.Native.vcxproj` (and `.filters`); CMake globs them automatically.

`Stdafx.h` is the precompiled header that pulls in MagickCore/MagickWand plus several `*-private.h` headers. On Windows it also emits the `#pragma comment(lib, ...)` line for every static delegate library via `MAGICK_NATIVE_LINK_LIB`; this is the list to update when ImageMagick gains or drops a dependency (the CMake side has the matching `LIB*` variables).

### Export and naming contract with Magick.NET

Magick.NET's source generator derives the P/Invoke `EntryPoint` from the managed class and member name, so the C names are not free-form:

- Function names are `<ManagedClass>_<Member>`; properties are `<Class>_<Name>_Get` / `<Class>_<Name>_Set`; instance lifetime is `<Class>_Create` / `<Class>_Dispose`.
- Every export is declared `MAGICK_NATIVE_EXPORT` (`__declspec(dllexport)`, `visibility("default")`, or `EMSCRIPTEN_KEEPALIVE` depending on platform).
- Functions that can fail take a trailing `ExceptionInfo **exception` and wrap the ImageMagick call in `MAGICK_NATIVE_GET_EXCEPTION;` ... `MAGICK_NATIVE_SET_EXCEPTION;` (from `Exceptions/MagickExceptionHelper.h`). GET opens a scope and acquires `exceptionInfo`; SET hands it to the caller only when a severity is set, otherwise destroys it, and closes the scope. To raise your own error use `MAGICK_NATIVE_RAISE_EXCEPTION(OptionError, "message")` and return before SET (see `MagickImage_Morphology`). On the managed side this corresponds to a `[Throws]` partial method.
- Operations that produce a new image return `Image *` and leave the input untouched; in-place operations return `void`. Channel-aware operations take `const size_t channels` and bracket the call with `SetChannelMask(instance, channels)` and either `RestoreChannelMask(instance)` (in-place) or `RestoreChannelMasks(instance, image)` (new image). These macros (top of `MagickImage.c`; a list-aware variant lives in `MagickImageCollection.c`) open/close a scope just like the exception macros, so keep the nesting order: GET_EXCEPTION, SetChannelMask, call, Restore..., SET_EXCEPTION.
- Enums cross the boundary as `size_t` and are cast to the ImageMagick enum inside; sizes are `size_t`/`ssize_t`; booleans are `MagickBooleanType`. Memory allocated by ImageMagick and returned to .NET is released through `MagickMemory_Relinquish`.

A typical change (see the recent "Added channel argument to ..." commits) touches exactly the `.h` declaration and the `.c` definition, and then needs a matching binding change in Magick.NET plus a new Magick.Native release before Magick.NET can use it. Keep header and source signatures in sync.

### Style

`.clang-format` and `.editorconfig` are authoritative: 2-space indentation, Allman braces on functions and control statements, no column limit, no include sorting, space after C-style casts, preprocessor `#` indented inside blocks. Local variables follow the ImageMagick house style (type on its own line, names indented on the next line, one declaration block at the top of the function). Null checks are written with an explicit typed cast, e.g. `if (image == (Image *) NULL)`. Every file starts with the two-line copyright/Apache-2.0 comment from `Copyright.txt`. Functions in a `.c`/`.h` are kept alphabetical after the `Create`/`Dispose` pair and the `_Get`/`_Set` property block.

### Resources

`src/Magick.Native/Resources/<Configuration>/<arch>/Magick.Native.rc` are tiny per-configuration Windows version-resource stubs (UTF-16) that define `NATIVE_CONFIGURATION`, `NATIVE_QUANTUM` and `NATIVE_PLATFORM` and include the shared `.rc`; add one when adding a new configuration. `Resources/xml/policy.xml` and `log.xml` ship in the Windows resources artifact and are embedded into the WASM constants.

## Releases

Releases are cut by manually dispatching the Main workflow. The version is a timestamp (`YYYY.MDD.HMM`), a tag is pushed, all per-platform zips plus the metadata (`NOTICE.txt`, `libraries.md` listing the bundled library versions) are attached to a GitHub release, and the WASM package is published to GitHub Packages. Magick.NET then pins that version string in its `Magick.Native.version` file.
