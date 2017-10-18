# MSVCBuildICU

Build ICU with Microsoft Visual Studio.

## Project

The CMakeLists.txt file was copied and adapted from [this gist](https://gist.github.com/jcfr/6030240) that
compiles OpenSSL. The steps to compile ICU with MS Visual Studio were found [here](https://wiki.qt.io/Compiling-ICU-with-MSVC).

This project generates an archive (`tar.gz`). The archive contains the `Debug` and the `Release`
install tree of `ICU` that is required to build DCMTK in ITK. This only includes the `include` folder
and two libraries: `icuuc` and `icudt`. These libraries are compiled statically. These
libraries are originally prefixed with `s` to show that it is a static version of the
library, but are renamed as part of the packaging process (FindICU.cmake does not look
for the prefixed library names).

## Usage

* Configure and generate project with CMake on Windows.
* Open Visual Studio, and compile `ALL_BUILD`.
* Upload the generated archive to [Midas](https://midas3.kitware.com/midas/community/12).

## To do

* Automatically upload generated files.
* Remove compilation error messages.
