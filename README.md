# Sinter

[![Coverage Status](https://coveralls.io/repos/github/source-academy/sinter/badge.svg?branch=master)](https://coveralls.io/github/source-academy/sinter?branch=master)

Name etymology: <strong>S</strong>VML <strong>inter</strong>preter.

This is an implementation of the Source Virtual Machine Language intended for microcontroller platforms like an Arduino. We follow the [Source VM specification](https://github.com/source-academy/js-slang/wiki/SVML-Specification) as in the js-slang wiki. Use this VM with the [reference compiler](https://github.com/source-academy/js-slang/blob/master/src/vm/svmc.ts).

For implementation details, see [here](vm/docs/impl.md).

## Directory layout

- `vm`: The actual VM library.
- `vm_test`: Some scripts to aid with CI testing.
- `runner`: A simple runner to run programs from the CLI.
- `test_programs`: SVML test programs that have been manually verified to be correct, as well as expected output for automated tests.
- `devices`: Some examples for using Sinter on various embedded platforms.

## Usage notes

Sinter implements most of Source §3, except:

- Numbers are single-precision floating points. This means that
  `16777216 + 1 === 16777216`.
- The following primitives are not supported:
  - list_to_string
  - parse_int
  - runtime
  - prompt
  - stringify

Usage recommendations:

- Treat arrays like C arrays, rather than JavaScript arrays (which are actually
  maps). Sinter does not (yet) have optimisations for sparse arrays.

## Use it on a device

Sinter is a C library. For examples on how to use Sinter, see [the CLI
runner](runner/src/runner.c), [the Arduino sketch
example](devices/arduino/arduino.ino), or the [ESP32
example](devices/esp32/src/main.c). There is also a [WASM example](devices/wasm).

To create an Arduino library zip, run the script
[`make_arduino_lib.sh`](make_arduino_lib.sh). You can configure the Arduino
library by unzipping the zip and modifying
[`sinter_config.h`](include/sinter_config.h).

## Build locally

We use the CMake build system. Note: a compiler that supports C11 is _required_.
This excludes MSVC.

```
# cd into the root of the repository, then:
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j8
make test
runner/runner ../test_programs/hello_world.svm
```

### Compiling your own programs

Use the [SVML compiler CLI utility in js-slang](https://github.com/source-academy/js-slang/blob/master/src/vm/svmc.ts) to compile programs for testing. (A real deployment of Sinter would integrate the compiler in js-slang directly instead.)

Alternatively, you could also try the [web demo](https://angelsl.github.io/sinter/), which uses Sinter compiled to WASM.

For convenience, we have included a NPM package that exposes the CLI utility.

Try it out:

```
# Make sure you have built the test runner from above.
# Then, from the root of the repository:
cd tools/compiler
yarn install
echo 'display("Hello world");' > myprogram.js
node svmc.js myprogram.js
../../build/runner/runner myprogram.svm
# (or wherever the test runner binary is)
```

### CMake configuration

Some configuration is available via CMake defines:

- `CMAKE_BUILD_TYPE`: controls the build type; defaults to `Debug`

  - `Debug`: assertions are enabled; some extra checks are enabled; `-Og` optimisation level
  - `Release`: assertions are disabled; `-O2` optimisation level

- `SINTER_HEAP_SIZE`: size in bytes of the statically-allocated heap; defaults
  to `0x10000` i.e. 64 KB

- `SINTER_STACK_ENTRIES`: size in stack entries of the statically-allocated
  stack; defaults to `0x200` i.e. 512

- `SINTER_DISABLE_CHECKS`: if `1`, disables certain safety checks in the runtime
  e.g. stack over/underflow checks; defaults to unset (i.e. safety checks are
  performed)

- `SINTER_DEBUG_LOGLEVEL`: controls the debug output level; defaults to `0`

  - `0`: all debug output is disabled.
  - `1`: prints reasons for most faults/errors
  - `2`: traces every instruction executed and every push on/pop off the stack

  This is available regardless of `CMAKE_BUILD_TYPE`.

  When deploying on an actual microcontroller, you will likely want to use `0`.
  `1` and `2` requires some implementation of `fprintf` and `stderr`. (This may
  be relaxed in future so the library consumer can provide a logging function
  instead.)

- `SINTER_DEBUG_ABORT_ON_FAULT`: if `1`, raises an assertion failure when a
  fault occurs. (Intended for use when debugging under e.g. GDB.) Defaults to
  unset.

- `SINTER_DEBUG_MEMORY_CHECK`: if `1`, does _a lot_ of checks at every
  instruction to verify the correctness of the heap linked list, freelist,
  stack, and reference counting. Note: this slows down execution severely.
  Defaults to unset.
