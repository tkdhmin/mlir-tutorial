# MLIR For Beginners

This is the code repository for a series of articles on the
[MLIR framework](https://mlir.llvm.org/) for building compilers.

## Articles

1.  [Build System (Getting Started)](https://jeremykun.com/2023/08/10/mlir-getting-started/)
2.  [Running and Testing a Lowering](https://jeremykun.com/2023/08/10/mlir-running-and-testing-a-lowering/)
3.  [Writing Our First Pass](https://jeremykun.com/2023/08/10/mlir-writing-our-first-pass/)
4.  [Using Tablegen for Passes](https://jeremykun.com/2023/08/10/mlir-using-tablegen-for-passes/)
5.  [Defining a New Dialect](https://jeremykun.com/2023/08/21/mlir-defining-a-new-dialect/)
6.  [Using Traits](https://jeremykun.com/2023/09/07/mlir-using-traits/)
7.  [Folders and Constant Propagation](https://jeremykun.com/2023/09/11/mlir-folders/)
8.  [Verifiers](https://jeremykun.com/2023/09/13/mlir-verifiers/)
9.  [Canonicalizers and Declarative Rewrite Patterns](https://jeremykun.com/2023/09/20/mlir-canonicalizers-and-declarative-rewrite-patterns/)
10. [Dialect Conversion](https://jeremykun.com/2023/10/23/mlir-dialect-conversion/)
11. [Lowering through LLVM](https://jeremykun.com/2023/11/01/mlir-lowering-through-llvm/)
12. [A Global Optimization and Dataflow Analysis](https://jeremykun.com/2023/11/15/mlir-a-global-optimization-and-dataflow-analysis/)


## Appendix for Articles (Tips from dhmin)
### Chapter 2: Running and Testing a Lowering
#### Build for examples programs
In the article, the examples programmed by MLIR are described. The example is to cout the leading zeros of a 32-bit interger digit.

1) Create `ctlz.mlir` file with the following content.

```llvm
func.func @main(%arg0: i32) -> i32 {
  %0 = func.call @my_ctlz(%arg0) : (i32) -> i32
  func.return %0 : i32
}
func.func @my_ctlz(%arg0: i32) -> i32 {
  %c32_i32 = arith.constant 32 : i32
  %c0_i32 = arith.constant 0 : i32
  %0 = arith.cmpi eq, %arg0, %c0_i32 : i32
  %1 = scf.if %0 -> (i32) {
    scf.yield %c32_i32 : i32
  } else {
    %c1 = arith.constant 1 : index
    %c1_i32 = arith.constant 1 : i32
    %c32 = arith.constant 32 : index
    %c0_i32_0 = arith.constant 0 : i32
    %2:2 = scf.for %arg1 = %c1 to %c32 step %c1 iter_args(%arg2 = %arg0, %arg3 = %c0_i32_0) -> (i32, i32) {
      %3 = arith.cmpi slt, %arg2, %c0_i32 : i32
      %4:2 = scf.if %3 -> (i32, i32) {
        scf.yield %arg2, %arg3 : i32, i32
      } else {
        %5 = arith.addi %arg3, %c1_i32 : i32
        %6 = arith.shli %arg2, %c1_i32 : i32
        scf.yield %6, %5 : i32, i32
      }
      scf.yield %4#0, %4#1 : i32, i32
    }
    scf.yield %2#1 : i32
  }
  func.return %1 : i32
}
```
2) Run mlir-opt to the source code using user-desired options like the following:

```bash
$ ./build/bin/mlir-opt --convert-scf-to-cf --convert-func-to-llvm --convert-arith-to-llvm ctlz.mlir -o ctlz.llvm.mlir
```

3) Translate the output into llvmir.

```bash
$ ./build/bin/mlir-translate --mlir-to-llvmir ctlz.llvm.mlir > ctlz.ll
```

4) Fix the error of the `ctlz.ll` encountered during clang compilation

```llvm
"""ctlz.ll"""
declare i8* @malloc(i64)

declare void @free(i8)
```

5) Run clang to create a binary executable
```bash
$ clang ctlz.ll -o ctlz
```

### Chapter 3: MLIR Writing our first pass
#### Build for example of `Transform` pass usage

In the article, the loop unrolling provided by `affine` dialect is the example of the usage of pass infrastructure.

1. Clarify the location of the following file:
```llvm
// location: mlir-tutorial/tests/affine_loop_unroll.mlir
func.func @test_single_nested_loop(%buffer: memref<4xi32>) -> (i32) {
  %sum_0 = arith.constant 0 : i32
  // CHECK-LABEL: test_single_nested_loop
  // CHECK-NOT: affine.for
  %sum = affine.for %i = 0 to 4 iter_args(%sum_iter = %sum_0) -> i32 {
    %t = affine.load %buffer[%i] : memref<4xi32>
    %sum_next = arith.addi %sum_iter, %t : i32
    affine.yield %sum_next : i32
  }
  return %sum : i32
}
```

2. Go to the `build/tools`.
```bash
cd build/tools
```

3. Execute the `tutorial-opt` binary with the `-affine-full-unroll` option.
```bash
./tutorial-opt ../../tests/affine_loop_unroll.mlir -affine-full-unroll -o output.mlir
```

4. Check the `output.mlir` where the loop has been unrolled.


## Bazel build

Bazel is one of two supported build systems for this tutorial. The other is
CMake. If you're unfamiliar with Bazel, you can read the tutorials at
[https://bazel.build/start](https://bazel.build/start). Familiarity with Bazel
is not required to build or test, but it is required to follow the articles in
the tutorial series and explained in the first article,
[Build System (Getting Started)](https://jeremykun.com/2023/08/10/mlir-getting-started/).
The CMake build is maintained, but was added at article 10 (Dialect Conversion)
and will not be explained in the articles.

### Prerequisites

Install Bazelisk via instructions at
[https://github.com/bazelbuild/bazelisk#installation](https://github.com/bazelbuild/bazelisk#installation).
This should create the `bazel` command on your system.

You should also have a modern C++ compiler on your system, either `gcc` or
`clang`, which Bazel will detect.


### Build and test

Run

```bash
bazel build ...:all
bazel test ...:all
```

## CMake build

CMake is one of two supported build systems for this tutorial. The other is
Bazel. If you're unfamiliar with CMake, you can read the tutorials at
[https://cmake.org/getting-started/](https://cmake.org/getting-started/). The
CMake build is maintained, but was added at article 10 (Dialect Conversion) and
will not be explained in the articles.

You can build LLVM and MLIR only with the following few steps.
### Step 0: Prerequisites

*   Make sure you have installed everything needed to build LLVM
    https://llvm.org/docs/GettingStarted.html#software
*   For this recipe Ninja is used so be sure to have it as well installed
    https://github.com/ninja-build/ninja/wiki/Pre-built-Ninja-packages
* Pybind
```bash
$ pip install pybind11
```
* protoc
```bash
$ git clone https://github.com/protocolbuffers/protobuf.git
$ cd protobuf
$ git checkout v3.20.0
$ git submodule update --init --recursive
$ mkdir build_source && cd build_source
$ cmake ../cmake -Dprotobuf_BUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_INSTALL_SYSCONFDIR=/etc -DCMAKE_POSITION_INDEPENDENT_CODE=ON -Dprotobuf_BUILD_TESTS=OFF -DCMAKE_BUILD_TYPE=Release
$ make -j$(nproc)
$ sudo make install
```
* g++9
```bash
$ sudo apt-get install gcc-9 g++-9
$ sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-9 90 # Adjust the your priority value
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 90 # Adjust the your priority value
$ g++ --version # Check whether the g++ version is 9 series.
```

### Step 1: Checking out the code
Checkout the tutorial including the LLVM dependency (submodules). Don't miss hte `--recurse-submodules` option

```bash
$ git clone --recurse-submodules https://github.com/j2kun/mlir-tutorial.git
$ cd mlir-tutorial
```

### Step 2: Build LLVM with MLIR
Note that the build script below is suitable for the ubuntuOS environment with ninja building system.
```bash
#!/bin/sh

BUILD_SYSTEM=Ninja
BUILD_TAG=ninja
THIRDPARTY_LLVM_DIR=/home/dhmin/mlir-tutorial/externals/llvm-project # Make sure that this path is valid to your env.
BUILD_DIR=$THIRDPARTY_LLVM_DIR/build
INSTALL_DIR=$THIRDPARTY_LLVM_DIR/install

mkdir -p $BUILD_DIR
mkdir -p $INSTALL_DIR

cd $BUILD_DIR

cmake ../llvm -G $BUILD_SYSTEM \
      -DCMAKE_CXX_COMPILER="/usr/bin/g++" \
      -DCMAKE_C_COMPILER="/usr/bin/gcc" \
      -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
      -DLLVM_LOCAL_RPATH=$INSTALL_DIR/lib \
      -DLLVM_PARALLEL_COMPILE_JOBS=7 \
      -DLLVM_PARALLEL_LINK_JOBS=1 \
      -DLLVM_BUILD_EXAMPLES=OFF \
      -DLLVM_INSTALL_UTILS=ON \
      -DCMAKE_OSX_ARCHITECTURES="$(uname -m)" \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_ASSERTIONS=ON \
      -DLLVM_CCACHE_BUILD=OFF \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
      -DLLVM_ENABLE_PROJECTS='mlir'

cmake --build . --target check-mlir
```

### Step 3: Prepare CMakeListst.txt

```bash
cmake_minimum_required(VERSION 3.20.0)

project(mlir-tutorial LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 17 CACHE STRING "C++ standard to conform to")
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(BUILD_DEPS ON)

# set(CMAKE_PREFIX_PATH "/home/dhmin/mlir-tutorial/externals/llvm-project/install")
find_package(MLIR REQUIRED CONFIG)

message(STATUS "Using MLIRConfig.cmake in: ${MLIR_DIR}")
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

set(MLIR_BINARY_DIR ${CMAKE_BINARY_DIR})

include(AddLLVM)
include(TableGen)

list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
include(AddMLIR)
include_directories(${LLVM_INCLUDE_DIRS})
include_directories(${MLIR_INCLUDE_DIRS})
include_directories(${PROJECT_SOURCE_DIR})
include_directories(${PROJECT_SOURCE_DIR}/externals/llvm-project)
include_directories(${PROJECT_BINARY_DIR})

message(STATUS "Fetching or-tools...")
include(FetchContent)
FetchContent_Declare(
  or-tools
  GIT_REPOSITORY https://github.com/google/or-tools.git
  GIT_TAG        v9.0
)
FetchContent_MakeAvailable(or-tools)
message(STATUS "Done fetching or-tools")

add_subdirectory(tests)
add_subdirectory(tools)
add_subdirectory(lib)
```

### Step 4: Build mlir-tutorial
Make a build and create the script by using the following code in the build directory. Please make sure that the LLMV_BUILD_DIR is valid to your environment.

```bash
BUILD_SYSTEM="Ninja"
LLVM_BUILD_DIR="/home/dhmin/mlir-tutorial/externals/llvm-project/build"

cmake -G $BUILD_SYSTEM .. \
    -DCMAKE_C_COMPILER=/usr/bin/gcc \
    -DCMAKE_CXX_COMPILER=/usr/bin/g++ \
    -DLLVM_DIR="$LLVM_BUILD_DIR/lib/cmake/llvm" \
    -DMLIR_DIR="$LLVM_BUILD_DIR/lib/cmake/mlir" \
    -DBUILD_DEPS="ON" \
    -DBUILD_SHARED_LIBS="OFF" \
    -DCMAKE_BUILD_TYPE=Debug \
    -DCMAKE_CXX_STANDARD=17
```

### Building dependencies ([Original code from Source](https://github.com/j2kun/mlir-tutorial))

Note: The following steps are suitable for macOs and use ninja as building
system, they should not be hard to adapt for your environment.

*Build LLVM/MLIR*

```bash
#!/bin/sh

BUILD_SYSTEM=Ninja
BUILD_TAG=ninja
THIRDPARTY_LLVM_DIR=$PWD/externals/llvm-project
BUILD_DIR=$THIRDPARTY_LLVM_DIR/build
INSTALL_DIR=$THIRDPARTY_LLVM_DIR/install

mkdir -p $BUILD_DIR
mkdir -p $INSTALL_DIR

pushd $BUILD_DIR

cmake ../llvm -G $BUILD_SYSTEM \
      -DCMAKE_CXX_COMPILER="$(xcrun --find clang++)" \
      -DCMAKE_C_COMPILER="$(xcrun --find clang)" \
      -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
      -DLLVM_LOCAL_RPATH=$INSTALL_DIR/lib \
      -DLLVM_PARALLEL_COMPILE_JOBS=7 \
      -DLLVM_PARALLEL_LINK_JOBS=1 \
      -DLLVM_BUILD_EXAMPLES=OFF \
      -DLLVM_INSTALL_UTILS=ON \
      -DCMAKE_OSX_ARCHITECTURES="$(uname -m)" \
      -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_ASSERTIONS=ON \
      -DLLVM_CCACHE_BUILD=ON \
      -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
      -DLLVM_ENABLE_PROJECTS='mlir' \
      -DDEFAULT_SYSROOT="$(xcrun --show-sdk-path)" \
      -DCMAKE_OSX_SYSROOT="$(xcrun --show-sdk-path)"

cmake --build . --target check-mlir

popd
```

### Build and test ([Original code from Source](https://github.com/j2kun/mlir-tutorial))

```bash
#!/bin/sh

BUILD_SYSTEM="Ninja"
BUILD_DIR=./build-`echo ${BUILD_SYSTEM}| tr '[:upper:]' '[:lower:]'`

rm -rf $BUILD_DIR
mkdir $BUILD_DIR
cd $BUILD_DIR

LLVM_BUILD_DIR=externals/llvm-project/build
cmake -G $BUILD_SYSTEM .. \
    -DLLVM_DIR="$LLVM_BUILD_DIR/lib/cmake/llvm" \
    -DMLIR_DIR="$LLVM_BUILD_DIR/lib/cmake/mlir" \
    -DBUILD_DEPS="ON" \
    -DBUILD_SHARED_LIBS="OFF" \
    -DCMAKE_BUILD_TYPE=Debug

cd ..

cmake --build $BUILD_DIR --target MLIRAffineFullUnrollPasses
cmake --build $BUILD_DIR --target MLIRMulToAddPasses
cmake --build $BUILD_DIR --target MLIRNoisyPasses
cmake --build $BUILD_DIR --target mlir-headers
cmake --build $BUILD_DIR --target mlir-doc
cmake --build $BUILD_DIR --target tutorial-opt
cmake --build $BUILD_DIR --target check-mlir-tutorial
```
