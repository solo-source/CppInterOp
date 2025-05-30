#########################
 Wasm Build Instructions
#########################

It should be noted that the wasm build of CppInterOp is still
experimental and subject to change.

************************************
 CppInterOp Wasm Build Instructions
************************************

This document first starts with the instructions on how to build a wasm
build of CppInterOp. Before we start it should be noted that unlike the
non wasm version of CppInterOp we currently only support the Clang-REPL
backend using llvm>19. We will first make folder to
build our wasm build of CppInterOp. This can be done by executing the
following command

.. code:: bash

   mkdir CppInterOp-wasm

Now move into this directory using the following command

.. code:: bash

   cd ./CppInterOp-wasm

To create a wasm build of CppInterOp we make use of the emsdk toolchain.
This can be installed by executing (we only currently support version
3.1.73)

.. code:: bash

   git clone https://github.com/emscripten-core/emsdk.git
   ./emsdk/emsdk install  3.1.73

and to activate the emsdk environment on Linux and osx execute 
(we are defining SYSROOT_PATH for use later)

.. code:: bash

   ./emsdk/emsdk activate 3.1.73
   source ./emsdk/emsdk_env.sh
   export SYSROOT_PATH=$PWD/emsdk/upstream/emscripten/cache/sysroot

and on Windows execute in Powershell

.. code:: powershell

   .\emsdk\emsdk activate 3.1.73
   .\emsdk\emsdk_env.ps1
   $env:PWD_DIR= $PWD.Path
   $env:SYSROOT_PATH="$env:EMSDK/upstream/emscripten/cache/sysroot"

Now clone the 20.x release of the LLVM project repository and CppInterOp
(the building of the emscripten version of llvm can be avoided by
executing micromamba install llvm -c
<https://repo.mamba.pm/emscripten-forge> and setting the LLVM_BUILD_DIR/$env:LLVM_BUILD_DIR
appropriately)

.. code:: bash

   git clone --depth=1 --branch release/20.x https://github.com/llvm/llvm-project.git
   git clone --depth=1 https://github.com/compiler-research/CppInterOp.git

Now move into the cloned llvm-project folder and apply the required patches. On Linux and osx this
executing

.. code:: bash

   cd ./llvm-project/
   git apply -v ../CppInterOp/patches/llvm/emscripten-clang20-*.patch

On Windows execute the following

.. code:: powershell

   cd .\llvm-project\
   cp -r ..\patches\llvm\emscripten-clang${{ matrix.clang-runtime }}*
   cp -r ..\patches\llvm\Windows-emscripten-clang${{ matrix.clang-runtime }}*
   git apply -v Windows-emscripten-clang20-1-CrossCompile.patch
   git apply -v emscripten-clang20-2-shift-temporary-files-to-tmp-dir.patch

We are now in a position to build an emscripten build of llvm by executing the following on Linux
and osx

.. code:: bash

   mkdir build
   cd build
   emcmake cmake -DCMAKE_BUILD_TYPE=Release \
                 -DLLVM_HOST_TRIPLE=wasm32-unknown-emscripten \
                 -DLLVM_ENABLE_ASSERTIONS=ON                        \
                 -DLLVM_TARGETS_TO_BUILD="WebAssembly" \
                 -DLLVM_ENABLE_LIBEDIT=OFF \
                 -DLLVM_ENABLE_PROJECTS="clang;lld" \
                 -DLLVM_ENABLE_ZSTD=OFF \
                 -DLLVM_ENABLE_LIBXML2=OFF \
                 -DCLANG_ENABLE_STATIC_ANALYZER=OFF \
                 -DCLANG_ENABLE_ARCMT=OFF \
                 -DCLANG_ENABLE_BOOTSTRAP=OFF \
                 -DCMAKE_CXX_FLAGS="-Dwait4=__syscall_wait4" \
                 -DLLVM_INCLUDE_BENCHMARKS=OFF                   \
                 -DLLVM_INCLUDE_EXAMPLES=OFF                     \
                 -DLLVM_INCLUDE_TESTS=OFF                        \
                 -DLLVM_ENABLE_THREADS=OFF                       \
                 -DLLVM_BUILD_TOOLS=OFF                          \
                 -DLLVM_ENABLE_LIBPFM=OFF                        \
                 -DCLANG_BUILD_TOOLS=OFF                         \
                 ../llvm
   emmake make libclang -j $(nproc --all)
   emmake make clangInterpreter clangStaticAnalyzerCore -j $(nproc --all)
   emmake make lldWasm -j $(nproc --all)

or executing

.. code:: powershell

   mkdir build
   cd build
   emcmake cmake -DCMAKE_BUILD_TYPE=Release `
                        -DLLVM_HOST_TRIPLE=wasm32-unknown-emscripten `
                        -DLLVM_ENABLE_ASSERTIONS=ON                        `
                        -DLLVM_TARGETS_TO_BUILD="${{ matrix.llvm_targets_to_build }}" `
                        -DLLVM_ENABLE_LIBEDIT=OFF `
                        -DLLVM_ENABLE_PROJECTS="${{ matrix.llvm_enable_projects }}" `
                        -DLLVM_ENABLE_ZSTD=OFF `
                        -DLLVM_ENABLE_LIBXML2=OFF `
                        -DCLANG_ENABLE_STATIC_ANALYZER=OFF `
                        -DCLANG_ENABLE_ARCMT=OFF `
                        -DCLANG_ENABLE_BOOTSTRAP=OFF `
                        -DCMAKE_CXX_FLAGS="-Dwait4=__syscall_wait4" `
                        -DLLVM_INCLUDE_BENCHMARKS=OFF                   `
                        -DLLVM_INCLUDE_EXAMPLES=OFF                     `
                        -DLLVM_INCLUDE_TESTS=OFF                        `
                        -DLLVM_ENABLE_THREADS=OFF                       `
                        -DLLVM_BUILD_TOOLS=OFF                          `
                        -DLLVM_ENABLE_LIBPFM=OFF                        `
                        -DCLANG_BUILD_TOOLS=OFF                         `
                        -G Ninja `
                        ..\llvm
   emmake ninja libclang clangInterpreter clangStaticAnalyzerCore lldWasm

on Windows. Once this finishes building we need to take note of where we built our llvm build.
This can be done by executing the following on Linux and osx

.. code:: bash

   export LLVM_BUILD_DIR=$PWD


and

.. code:: powershell

   $env:PWD_DIR= $PWD.Path
   $env:LLVM_BUILD_DIR="$env:PWD_DIR\llvm-project\build"


on Windows. We can move onto building the wasm version of CppInterOp. We will do
this within a Conda environment. We can achieve this by executing
(assumes you have micromamba installed and that your shell is
initialised for the micromamba install)

.. code:: bash

   cd ../../CppInterOp/
   micromamba create -f environment-wasm.yml --platform=emscripten-wasm32
   micromamba activate CppInterOp-wasm

You will also want to set a few environment variables. On Linux and osx you define them as follows

.. code:: bash

   export PREFIX=$CONDA_PREFIX
   export CMAKE_PREFIX_PATH=$PREFIX
   export CMAKE_SYSTEM_PREFIX_PATH=$PREFIX

and

.. code:: powershell

   $env:PREFIX="%CONDA_PREFIX%/envs/CppInterOp-wasm"
   $env:CMAKE_PREFIX_PATH=$env:PREFIX
   $env:CMAKE_SYSTEM_PREFIX_PATH=$env:PREFIX

on Windows. Now to build and test your Emscripten build of CppInterOp on Linux and osx execute the following

.. code:: bash

   mkdir build
   cd ./build/
   emcmake cmake -DCMAKE_BUILD_TYPE=Release    \
                 -DLLVM_DIR=$LLVM_BUILD_DIR/lib/cmake/llvm      \
                 -DLLD_DIR=$LLVM_BUILD_DIR/lib/cmake/lld     \
                 -DClang_DIR=$LLVM_BUILD_DIR/lib/cmake/clang     \
                 -DBUILD_SHARED_LIBS=ON                      \
                 -DCMAKE_FIND_ROOT_PATH_MODE_PACKAGE=ON            \
                 -DCMAKE_INSTALL_PREFIX=$PREFIX         \
                 -DSYSROOT_PATH=$SYSROOT_PATH                                   \
                 ../
   emmake make -j $(nproc --all) check-cppinterop

To build and test your Emscripten build of CppInterOp on Windows execute the following

.. code:: powershell

   emcmake cmake -DCMAKE_BUILD_TYPE=${{ env.BUILD_TYPE }}    `
                -DCMAKE_PREFIX_PATH="$env:PREFIX"                      `
                -DLLVM_DIR="$env:LLVM_BUILD_DIR\lib\cmake\llvm"        `
                -DLLD_DIR="$env:LLVM_BUILD_DIR\lib\cmake\lld"        `
                -DClang_DIR="$env:LLVM_BUILD_DIR\lib\cmake\clang"    `
                -DBUILD_SHARED_LIBS=ON                      `
                -DCODE_COVERAGE=${{ env.CODE_COVERAGE }}    `
                -DCMAKE_INSTALL_PREFIX="$env:PREFIX"      `
                -DCMAKE_FIND_ROOT_PATH_MODE_PACKAGE=ON            `
                -DLLVM_ENABLE_WERROR=On                      `
                -DSYSROOT_PATH="$env:SYSROOT_PATH"                     `
                ..\
   emmake make -j $(nproc --all) check-cppinterop

Assuming it passes all test you can install by executing the following. 

.. code:: bash

   emmake make -j $(nproc --all) install

## Xeus-cpp-lite Wasm Build Instructions

A project which makes use of the wasm build of CppInterOp is xeus-cpp.
xeus-cpp is a C++ Jupyter kernel. Assuming you are in the CppInterOp
build folder, you can build the wasm version of xeus-cpp by executing

.. code:: bash

   cd ../..
   git clone --depth=1 https://github.com/compiler-research/xeus-cpp.git
   cd ./xeus-cpp
   mkdir build
   cd build
   emcmake cmake \
           -DCMAKE_BUILD_TYPE=Release                                     \
           -DCMAKE_PREFIX_PATH=$PREFIX                                    \
           -DCMAKE_INSTALL_PREFIX=$PREFIX                                 \
           -DXEUS_CPP_EMSCRIPTEN_WASM_BUILD=ON                            \
           -DCMAKE_FIND_ROOT_PATH_MODE_PACKAGE=ON                         \
           -DSYSROOT_PATH=$SYSROOT_PATH                                   \
           ..
   emmake make -j $(nproc --all) install

To build Jupyter Lite website with this kernel locally that you can use
for testing execute the following

.. code:: bash

   cd ../..
   micromamba create -n xeus-lite-host jupyterlite-core -c conda-forge
   micromamba activate xeus-lite-host
   python -m pip install jupyterlite-xeus jupyter_server
   jupyter lite build --XeusAddon.prefix=$PREFIX --contents xeus-cpp/notebooks/xeus-cpp-lite-demo.ipynb

Once the Jupyter Lite site has built you can test the website locally by
executing

.. code:: bash

   jupyter lite serve --XeusAddon.prefix=$PREFIX
