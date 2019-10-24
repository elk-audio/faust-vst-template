## Faust VST2 template

Template for a CMake project that builds Faust .dsp files into headless Linux VST 2 plugins using a cross-compilation toolchain.

Originally developed and tested for usage with Elk Audio OS but could be used in similar contexts as well.

Inspired by the `faust2faustvst` script by A. Graef included in the standard Faust distribution. The rationale behind this CMake template is to have something that works in a cross-compilation environment. CMake is supported by Yocto/bitbake and similar tools and it is much easier to maintain in this respect to hand-written build scripts.

# Usage

Requirements:

  * Faust installed on your host machine (not included in Elk Audio OS toolchain). Tested with Faust 2.15.11
  * Cross-compilation environment with g++ version with C++17 support and Boost libraries

Put the .dsp files you want to compile in the `faust-code` subdirectory, or in another path specified by the CMake variable `FAUST_SOURCES_DIR`.

The VsT 2.4 SDK must be provided in order to build the plugins. The SDK can be copied into the root of the repository or to the path specified by the CMake variable `VST2_SDK_PATH`.

Then compile with standard CMake workflow, e.g.
``mkdir build && cd build``
``cmake ..``
``ccmake ..`` (to eventually change CMake options)
``make``

The other options available, i.e. those starting with `FAUST_*`, are the relevant ones (for a headless build) in the original `faust2faustvst` script.

---
Elk Audio OS
Copyright (C) 2019 Elk
Stockholm

---
Faust
Copyright (C) 2003-2019 GRAME, Centre National de Creation Musicale
https://faust.grame.fr/doc/index.html

