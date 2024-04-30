# cling build windows
How to build a working version of cling C++ interpreter version 1.0 with MSVC on Windows 10/11.


### Install build environment
Install Visual Studio 2022 with MSVC.  
Install Python3 from the Windows app store so it ends up in PATH in cmd.  


### Clone repos
`git clone https://github.com/root-project/cling` into folder `C:\dev\cling\ `.  
`git clone https://github.com/root-project/llvm-project` into folder `C:\dev\llvm-project\ `. This one is HUGE and SLOW.  
Switch branch in `llvm-project` to `cling-latest` (or version of choice).  


### Setup build environment
Open up a `cmd` command prompt and keep it open.  
Create and cd into build folder:
```
mkdir C:\dev\cling-build\
cd C:\dev\cling-build\
```
Init build environment:
```
C:\"Program Files"\"Microsoft Visual Studio"\2022\Community\VC\Auxiliary\Build\vcvarsall.bat x64 10.0.22621.0 -vcvars_ver=14.37.32822
```
For finding and executing `vcvarsall.bat` on your local machine:  
Lookup your local Visual Studio install path: `C:\"Program Files (x86)"\"Microsoft Visual Studio"\Installer\vswhere.exe`.  
Lookup your local `winsdk_version` (10.0.22621.0 above) via `regedit` key `Computer\HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Microsoft\Microsoft SDKs\Windows\ `.  
Lookup your local `vcvars_ver` (14.37.32822 above) in folder `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\ `.  


### Build
In the `cmd` command prompt from the previous step execute the following:
```
cmake -G Ninja -DCMAKE_CXX_STANDARD=17 -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX="./install/" -DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc -DLLVM_EXTERNAL_PROJECTS=cling -DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/ -DLLVM_ENABLE_PROJECTS=clang -DLLVM_TARGETS_TO_BUILD="host;NVPTX" -DLLVM_BUILD_TOOLS=OFF -DCLANG_BUILD_TOOLS=OFF -Dbuiltin_llvm=ON -Dbuiltin_clang=ON ../llvm-project/llvm/
```
```
cmake --build . --target install
```


### Package
Navigate to `cling-build\install\ ` folder.  
Copy the following files to where you want your cling installation to reside (preserve folder structure):
```
.\bin\cling.exe
.\include\cling\*
.\lib\clang\16\include\*
```
Consider adding the final install path to PATH.  
And that's it! Enjoy!


## The journey to discovering the build config
In build folder try running the following command:  
```
cmake
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;nvptx"
	../llvm-project/llvm
```
Issues with nvptx.  
Try compiling with clang-cl instead:
```
cmake
	-DCMAKE_C_COMPILER=clang-cl.exe
	-DCMAKE_CXX_COMPILER=clang-cl.exe
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;nvptx"
	../llvm-project/llvm
```
Still issues with nvptx in both cases.  
Try what the build log suggests regarding nvptx being experimental:
```
cmake -DLLVM_EXTERNAL_PROJECTS=cling -DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/ -DLLVM_ENABLE_PROJECTS=clang -DLLVM_TARGETS_TO_BUILD="host" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="nvptx" ../llvm-project/llvm
cmake -DLLVM_EXTERNAL_PROJECTS=cling -DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/ -DLLVM_ENABLE_PROJECTS=clang -DLLVM_TARGETS_TO_BUILD="host;nvptx" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="nvptx" ../llvm-project/llvm
cmake --build . --config Release --target install
```
Still issues with nvptx.  
Oh. nvptx is case sensitive and must be NVPTX:
```
cmake
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	../llvm-project/llvm
cmake --build .
```
**It compiles!**  
But executing `cling.exe` outputs warnings/errors so it's not really usable.  
Try build release build, with generator ninja, and exclude some superfluous llvm components with `-DLLVM_BUILD_TOOLS=OFF`.
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Release
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	../llvm-project/llvm
cmake --build .
```
Still issues with executing `cling.exe`.  
3811 -> 3410 files to build when disabling `LLVM_BUILD_TOOLS`. So somewhat faster builds.  
Oh, apparently `-DCLANG_BUILD_TOOLS=OFF` is undocumented but exists as of late. Let's try it:  
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Release
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	../llvm-project/llvm
cmake --build .
```
3410 -> 3355 files to build when disabling `CLANG_BUILD_TOOLS`.  
Try compiling with clang-cl instead:
```
cmake
	-G Ninja
	-DCMAKE_C_COMPILER=clang-cl.exe
	-DCMAKE_CXX_COMPILER=clang-cl.exe
	-DCMAKE_BUILD_TYPE=Release
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	../llvm-project/llvm
cmake --build .
```
Compilation errors with clang-cl: `FAIL Interpreter.cpp(586,58): error: use of undeclared identifier '_CxxThrowException'`.  
The reason is clang-cl defines `_MSC_VER` but `_CxxThrowException` is missing.  
``_CxxThrowException`` seems to only exist with MSVC so I guess we're forced to use MSVC.  
Seems like `-DLLVM_HOST_TRIPLE=x86_64` might be a required flag so let's try it next:
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Release
	-DLLVM_HOST_TRIPLE=x86_64
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	../llvm-project/llvm
cmake --build .
```
So it builds and executes `cling.exe` without any warnings.  
But it seems to fail linking the standard lib since includes such as these fail: `#include <stdio.h>` and `#include <iostream>`.  
Let's try build in debug instead of release:
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Debug
	-DLLVM_HOST_TRIPLE=x86_64
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	../llvm-project/llvm
cmake --build .
```
Same issue remain.  
What about `-DLLVM_HOST_TRIPLE=x86_64`? Should it be `x86_64-pc-windows-msvc`? Or perhaps `x86_64-pc-win64`?  
Reading the docs it's most likely `x86_64-pc-win32-msvc` we need:
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Debug
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	../llvm-project/llvm
cmake --build .
```
**It runs**!   
Let's try build with Release this time:
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Release
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	../llvm-project/llvm
cmake --build .
```
Still runs!  
Significantly faster runtime when executing `#include` instructions.  
Do we have to do `cmake --build .` or can we do just `cmake --build . --target cling` for faster build times and smaller build results?
```
cmake --build . --target cling
```
3355 -> 2647 files to build but: `ERROR in cling::CIFactory::createCI(): resource directory C:/dev/cling-build\lib\clang\16 not found!`  
But that suspiciously looking absolute path is a bit worrying: *are builds not portable?*  
Trying a working build on another computer confirms the suspicion: *build are not portable!*  
So let's try add  `-DCMAKE_INSTALL_PREFIX="./install/"` and instead of just `cmake build .` run `cmake --build . --target install`:
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Release
	-DCMAKE_INSTALL_PREFIX="./install/"
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	../llvm-project/llvm/
cmake --build . --target install
```
Still not portable.  
Everything seems to be neatly packaged into the install folder but the `lib\clang\16` path is still an absolute path thus the build is not portable.  
Looking at LLVM_PATH usage in `cling/lib/Interpreter/CMakeLists.txt` suggests two unset variables `builtin_llvm` and `builtin_clang`.  
According to the ROOT project docs those two should supposedly be `ON` by default but apparently they're not.  
So let's try turning them on:
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=Release
	-DCMAKE_INSTALL_PREFIX="./install/"
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	-Dbuiltin_llvm=ON
	-Dbuiltin_clang=ON
	../llvm-project/llvm/
cmake --build . --target install
```
**It's portable!**  
By now it runs on other computers successfully.  
But the install folder is ~1.3 GB in size so not really that portable after all.  
File size of `cling.exe` alone is 70 MB.  
Let's try a `MinSizeRel` build:
```
cmake
	-G Ninja
	-DCMAKE_BUILD_TYPE=MinSizeRel
	-DCMAKE_INSTALL_PREFIX="./install/"
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	-Dbuiltin_llvm=ON
	-Dbuiltin_clang=ON
	../llvm-project/llvm/
cmake --build . --config MinSizeRel --target install
```
Now file size of `cling.exe` is down to 58 MB.  
I will benefit more from speed over size so let's not pursue that path any further.  
Let's see if building with the MSVC generator works as well and what build size we get:
```
cmake
	-G "Visual Studio 17 2022" -A x64 -Thost=x64
	-DCMAKE_BUILD_TYPE=Release
	-DCMAKE_INSTALL_PREFIX="./install/"
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	-Dbuiltin_llvm=ON
	-Dbuiltin_clang=ON
	../llvm-project/llvm/
cmake --build . --config Release --target install
```
MSVC generator works as well as Ninja.  
But the size of install folder and `cling.exe` remains unchanged.  
Let's try reducing the install folder by removing a bunch of files.  

**It's truly portable!**  
The build still works after reducing the install folder down to only the following items:
```
.\bin\cling.exe
.\include\cling\*
.\lib\clang\16\include\*
```
By now the install size is 76 MB of which `cling.exe` takes up 70 MB and the rest is .h files.  
So the only size we can significantly affect by now is the build size of `cling.exe`.  
Besides `MinSizeRel` trying a bunch of LLVM compilation flags turns out to not really affect size.  

Let's have a look at C++ version support next.  
Inspecting the custom LLVM project `llvm/CMakeLists.txt` tells us we build cling with LLVM version 16.0.6.  
That clang version defaults to C++17 but should have partial C++20 support.  
Executing `__cplusplus` in the cling REPL outputs `201703`, i.e. C++17.  
So if we want C++20 we should try build with `-DCMAKE_CXX_STANDARD=20`.  
The compiler immediately fires off a bunch of warnings we can suppress with `_SILENCE_ALL_CXX20_DEPRECATION_WARNINGS`.  
Suppress warnings by specifying `CMAKE_CXX_FLAGS_INIT` which prepends to `CMAKE_CXX_FLAGS` instead of overriding it.  
Final build config for C++20:
```
cmake
	-G Ninja
	-DCMAKE_CXX_STANDARD=20
	-DCMAKE_CXX_FLAGS_INIT="/D_SILENCE_ALL_CXX20_DEPRECATION_WARNINGS"
	-DCMAKE_BUILD_TYPE=Release
	-DCMAKE_INSTALL_PREFIX="./install/"
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	-Dbuiltin_llvm=ON
	-Dbuiltin_clang=ON
	../llvm-project/llvm/
cmake --build . --target install
```
**It runs with C++20**  
Executing `__cplusplus` in the cling REPL now outputs `202002` instead, i.e. C++20.  
But it turns out executing some common `#include` instructions is significantly slower (almost twice as slow).  
Trying to pinpoint the root cause I inspected some of the MSVC STL source files I directly and indirectly include.  
I found some ifdefs causing more includes if C++20 or above so it might be just that.  
I will benefit more from fast dev iterations so I will revert back to C++17 by keeping that build flag explicit.  
Final build config for C++17:
```
cmake
	-G Ninja
	-DCMAKE_CXX_STANDARD=17
	-DCMAKE_BUILD_TYPE=Release
	-DCMAKE_INSTALL_PREFIX="./install/"
	-DLLVM_HOST_TRIPLE=x86_64-pc-win32-msvc
	-DLLVM_EXTERNAL_PROJECTS=cling
	-DLLVM_EXTERNAL_CLING_SOURCE_DIR=../cling/
	-DLLVM_ENABLE_PROJECTS=clang
	-DLLVM_TARGETS_TO_BUILD="host;NVPTX"
	-DLLVM_BUILD_TOOLS=OFF
	-DCLANG_BUILD_TOOLS=OFF
	-Dbuiltin_llvm=ON
	-Dbuiltin_clang=ON
	../llvm-project/llvm/
cmake --build . --target install
```
So by that I guess we are done!  
If wanting smaller builds at the cost of speed add `-DCMAKE_CXX_FLAGS_INIT="/Os"`.  
