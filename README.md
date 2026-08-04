# FFmpeg Commercial SDK (Portable & Optimized)

[![Build FFmpeg SDK (Stable)](https://github.com/John-Giskhan/FFmpeg-dynamic-portable-build/actions/workflows/build_ffmpeg.yaml/badge.svg)](https://github.com/John-Giskhan/FFmpeg-dynamic-portable-build/actions/workflows/build_ffmpeg.yaml)
![License](https://img.shields.io/badge/License-LGPL_v3-blue.svg)
![Platform](https://img.shields.io/badge/Platform-Windows_x64-blue.svg)

This repository hosts automated, daily builds of **FFmpeg** explicitly optimized for commercial software development on Windows.

Unlike standard builds, these artifacts are **LGPL v3 compliant**, meaning they can be dynamically linked in proprietary/closed-source C++ applications without violating strict GPL rules.

---

## 🚀 Key Features

### ⚖️ Commercial Ready
* **LGPL v3 Compliant:** No GPL-only components (like `libx264`/`libx265`) are included.
* **Patent Safety:** Prioritizes hardware encoding (NVENC/AMF) and open standards (AV1/VP9/Opus) to minimize royalty risks.
* **MSVC Compatible:** Includes `.lib` import libraries for easy linking with Visual Studio and Clang-CL.

### ⚡ Hardware Accelerated
* **Nvidia:** NVENC (Encoding) & NVDEC (Decoding) enabled via license-safe headers.
* **AMD:** AMF (Advanced Media Framework) enabled.
* **Universal:** **Vulkan Video** enabled for cross-vendor hardware decoding and encoding.

### 📦 Truly Portable
* **Dependency Bundled:** All required third-party DLLs (Opus, libplacebo, SVT-AV1, dav1d, libvpx, shaderc, Zlib, etc.) are included in the `bin/` folder.
* **No Installation:** The end-user does not need MSYS2 or MinGW installed.

> [!IMPORTANT]
> **Vulkan runtime required.** `vulkan-1.dll` is the one dependency that is *not* bundled — it is provided by your GPU driver, and the Vulkan loader is designed to be used from the system rather than shipped alongside an application.
>
> `avfilter` links `libplacebo`, which imports `vulkan-1.dll` at load time, so **loading `avfilter` fails on a machine without a Vulkan runtime** (error `0xc0000135`). Any machine with current GPU drivers is fine; bare Windows Server images and GPU-less VMs may not be.

### 🏎️ Speed Optimized
* **AV1:** Includes `SVT-AV1` (Fastest CPU Encoder) + Hardware AV1 support (RTX 40-series/RDNA3).
* **Scaling:** Includes `scale_vulkan` and `libplacebo` (Vulkan) for zero-copy GPU resizing.

> [!NOTE]
> `scale_cuda` is **not** included. Its CUDA kernels require a device compiler (`--enable-cuda-nvcc` or `--enable-cuda-llvm`) that is not available on the build runner. Use `scale_vulkan` or `libplacebo` for GPU scaling.

---

## 🤖 How It Works

This repository uses a **GitHub Actions** workflow that runs daily:
1.  **Checks Upstream:** Queries the official FFmpeg repository for the latest **stable** tag (e.g., `n9.0`), falling back to the project's GitHub mirror if `git.ffmpeg.org` is unreachable.
2.  **Builds:** If a new version is found, it spins up a Windows environment, installs MSYS2, and compiles FFmpeg from source.
3.  **Bundles:** It automatically detects and copies all system dependencies (DLLs) using `ldd`.
4.  **Publishes:** The final SDK (Headers + Libs + DLLs) is uploaded to the **Releases** page.

---

## 📦 Integration Guide (CMake)

You do not need to use `git submodule`. We recommend using CMake's `FetchContent` to download the specific binary release. This keeps your git history clean and your build fast.

### 1. Update `CMakeLists.txt`

Add this block to the top of your project's CMake file.

**Note:** Pin the version tag you want. Because each release's asset is named after its version
(`ffmpeg-sdk-n9.0.zip`), a `releases/latest/download/...` URL breaks as soon as a new version ships —
so pin explicitly and bump it deliberately.

```cmake
cmake_minimum_required(VERSION 3.24)
project(MyMediaServer)

include(FetchContent)

# ------------------------------------------------------------------------------
# 1. Download FFmpeg SDK from GitHub Releases
# ------------------------------------------------------------------------------
set(FFMPEG_SDK_VERSION "n9.0")

FetchContent_Declare(
    ffmpeg_sdk
    URL "https://github.com/John-Giskhan/FFmpeg-dynamic-portable-build/releases/download/${FFMPEG_SDK_VERSION}/ffmpeg-sdk-${FFMPEG_SDK_VERSION}.zip"
    DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)
FetchContent_MakeAvailable(ffmpeg_sdk)

# Define the root path where CMake unzipped it
set(FFMPEG_ROOT "${ffmpeg_sdk_SOURCE_DIR}")

# ------------------------------------------------------------------------------
# 2. Setup Your Target
# ------------------------------------------------------------------------------
add_executable(MyMediaServer main.cpp)

# A. Include Headers
target_include_directories(MyMediaServer PRIVATE "${FFMPEG_ROOT}/include")

# B. Link Libraries
# We point to /lib where the MSVC-compatible .lib files live
target_link_directories(MyMediaServer PRIVATE "${FFMPEG_ROOT}/lib")

# Link against the core libraries
target_link_libraries(MyMediaServer PRIVATE
    avdevice
    avformat
    avfilter
    avcodec
    swscale
    swresample
    avutil
)

# ------------------------------------------------------------------------------
# 3. Deploy DLLs (Post-Build)
# ------------------------------------------------------------------------------
# This automatically copies all .dll files from the SDK to your .exe folder
if(WIN32)
    add_custom_command(TARGET MyMediaServer POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        "${FFMPEG_ROOT}/bin"
        "$<TARGET_FILE_DIR:MyMediaServer>"
        COMMENT "Deploying FFmpeg Runtime DLLs..."
    )
endif()