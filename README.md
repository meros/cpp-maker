# Maker

Maker is a small convention-based layer on top of CMake for C++ projects. You put each app, library or protocol in its own folder, write a few lines in a `makerfile`, and Maker creates the CMake targets, the include paths, the link dependencies and the Google Test executables for you. The goal was to make it fast to set up proof-of-concept and hobby projects that have several libraries and applications that depend on each other.

> **Status:** This is a personal project from 2013–2016. It is finished and not maintained. It was written for the CMake 2.8/3.x of that time; current CMake releases (4.x) reject its `cmake_minimum_required(VERSION 2.6)`, and the nRF51 toolchain file uses the deprecated `CMakeForceCompiler` module.

## How it works

A project has one top-level `CMakeLists.txt` that includes Maker and calls `maker_init`:

```cmake
cmake_minimum_required(VERSION 2.6)
include(../maker/maker.cmake)
maker_init(MyProject)
```

`maker_init` scans three folders under the project root. Every subfolder that contains a `makerfile` is a module, and the folder name is the module name:

| Folder                 | Module type | CMake target        |
| ---------------------- | ----------- | ------------------- |
| `apps/<name>/`         | app         | executable `app_<name>` |
| `libs/<name>/`         | lib         | static library `lib_<name>` |
| `protocols/<name>/`    | protocol    | library `protocol_<name>`, generated from `.proto` files with protobuf |

The `makerfile` is plain CMake that runs in the scope of the module. It uses two macros:

- `maker_module_set_sources(<files>...)` adds source files, relative to the module folder.
- `maker_module_depend(<name>)` adds a dependency. Maker looks up `<name>` as a lib in `libs/`, as a protocol in `protocols/`, and as an external library in `ext-libs/<name>.makerfile` (first in the project, then in `maker/ext-libs/`).

Other rules:

- A lib exposes the headers in its `include/` folder to the modules that depend on it.
- If a module has a `gtests/` folder, Maker compiles every `*.cpp` in it into a test executable `<target>_test`, links it with the module and Google Test 1.7.0 (bundled), and registers it with CTest.
- Maker prints debug messages during configuration. Set `-DMAKER_PRINT_DEBUG=OFF` to turn them off.

The bundled external libraries are `boost` (Boost ≥ 1.59, `system` component), `protobuf` and `nrfsdk` (the Nordic nRF5 SDK 11.0.0 for the nRF51822).

## Example

`testproject/` shows the main use cases:

```
testproject/
├── CMakeLists.txt
├── apps/
│   ├── app2/        makerfile, test.cpp
│   └── testapp/     makerfile, main.cpp, another.cpp
└── libs/
    ├── testlib/     makerfile, test.cpp, include/test.h, gtests/*.cpp
    └── testlib2/    makerfile, test2.cpp, include/test2.h
```

`apps/testapp/makerfile`:

```cmake
maker_module_depend(testlib)
maker_module_depend(boost)
maker_module_set_sources(main.cpp another.cpp)
```

`libs/testlib/makerfile`:

```cmake
maker_module_depend(testlib2)
maker_module_set_sources(include/test.h test.cpp)
```

Build it out of source:

```sh
mkdir testproject-build && cd testproject-build
cmake ../testproject
make
make test
```

This gives `app_app2`, `app_testapp`, `liblib_testlib.a`, `liblib_testlib2.a` and the test executable `lib_testlib_test`.

## Cross compilation (nRF51822)

The last work in 2016 added cross compilation for the Nordic nRF51822 (ARM Cortex-M0). `crossplatformproject/` has a `blinky` app that flashes an LED through a small `nrf51io` lib. The toolchain file expects GCC ARM Embedded 4.9 (2015q3) at a fixed path in `/usr/local`:

```sh
mkdir blinky-build && cd blinky-build
cmake -DCMAKE_TOOLCHAIN_FILE=../maker/platforms/nrf51822/toolchain.cmake ../crossplatformproject
make
```

This part was experimental. The last commit message is "Ninja is blinking!".

## Repository layout

- `maker/`: the build system (`maker.cmake`, `tools/`, `ext-libs/`), the bundled Google Test 1.7.0, and `platforms/nrf51822/` with the toolchain file, the linker script and a copy of the Nordic nRF5 SDK 11.0.0.
- `testproject/`: the example for the host compiler.
- `crossplatformproject/`: the nRF51822 example.

The [wiki](https://github.com/meros/cpp-maker/wiki/Your-first-build-with-maker) has a longer walkthrough of the first build.

## License

Maker is released under the [MIT License](LICENSE). The bundled third-party code keeps its own license: Google Test in `maker/googletest-release-1.7.0/LICENSE` and the Nordic SDK in `maker/platforms/nrf51822/SDK/licenses.txt`.
