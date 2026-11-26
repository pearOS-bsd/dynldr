<div align="center">

# dynldr

**A dynamic loader that runs Mac programs on Linux, with the host libraries behind it.**

C and C++ · 60,000 lines · BSD-3-Clause

</div>

---

dynldr loads Mach-O programs on Linux and gives them what they expect from macOS: Mach ports, bundles, code
signatures, the system calls, and a host for their windows, text, sound, video and Metal shaders.

Every function was rebuilt from published binaries, one at a time, and compared with the original. Today
**3,850 of 5,060 functions (76 %) compile to the same instructions** as the original. Where the programs were tested, they give the same
output as the original in black-box tests.

## Components

| | directory | what it does |
|---|---|---|
| **Loader** | `src/loader` | Maps Mach-O images, links them, and runs them. Includes the Mach broker (ports and messages between processes), the sandbox, signals, sockets, threads and Bonjour. |
| **AppKit host** | `src/appkit-host` | Everything a Mac program's user interface asks of the system: windows, drawing, input, menus, text and fonts, PDF, OpenGL, sound, cameras and hardware video decoding (H.264, HEVC, VP9). |
| **Metal** | `src/metal-translator`, `src/metal-msl` | Compiles Metal Shading Language to AIR (with clang 19) and translates metallibs and AIR to GLSL (with LLVM 19). |
| **WebKit host** | `src/webkit-host` | The bridge between Mac programs and a native WebKit. |
| **Disk images** | `src/hdiutil` | Creates and reads HFS+, ISO 9660, Joliet, UDF and all seven UDIF formats. LZFSE in `upstream/`. |
| **Packages** | `src/install-package` | Lists and installs `.pkg` installers. |
| **Gatekeeper** | `src/open`, `src/unsupported` | Checks code signatures before launch and explains why a program cannot run. |
| **Diagnostics** | `src/diagnose` | Reports what the machine's graphics stack can do. |
| **Session** | `src/login`, `src/authhelper` | Login and the privileged helper. |

## Building

The original binaries were built on Debian trixie with GCC 14. `docker/Dockerfile` sets up that environment, with the
package archive fixed at the day of the build. Build in it:

```sh
docker build -t dynldr-build docker
docker run --rm -it -v "$PWD:/w" dynldr-build sh -c '
  H="-g -O2 -fstack-protector-strong -fstack-clash-protection -fcf-protection -D_FORTIFY_SOURCE=2 -Wdate-time"
  cmake -S /w -B /w/build -G Ninja -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_C_FLAGS="$H" -DCMAKE_CXX_FLAGS="$H" -DCMAKE_EXE_LINKER_FLAGS=-Wl,-z,relro
  ninja -C /w/build'
```

Use the same flags. The code depends on them to give the same instructions as the original.

Without any options, CMake builds the loader, login, the helper, disk images and, when LLVM 19 is present, both Metal
libraries. The other components use the desktop toolkit (commit `7e25ddc`). Give CMake its source and libraries
to build them too:

| option | value |
|---|---|
| `TOOLKIT_SRC` | a checkout of the toolkit |
| `TOOLKIT_LIB` | the directory with the toolkit's shared libraries |
| `WEBKIT_INCLUDE` | the WebKit headers (WebKit host only) |

> [!CAUTION]
> `login` and `authhelper` handle credentials. Test them in a container. Never install them setuid from a development
> build.

## Licence

BSD-3-Clause, as the work it is derived from. See [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).
Third-party code in `upstream/` keeps its own licence; see [`upstream/PROVENANCE.md`](upstream/PROVENANCE.md).
