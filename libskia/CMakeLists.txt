# libskia - Skia 2D Graphics Library
# Built from source with architecture-specific optimizations
# Mirrors the GYP build from libskia.gyp

message(STATUS "Configuring libskia from source")

# ============================================================
# Architecture detection
# On macOS, CMAKE_OSX_ARCHITECTURES overrides CMAKE_SYSTEM_PROCESSOR
# (e.g., building x86_64 on an arm64 host)
# ============================================================
if(APPLE AND CMAKE_OSX_ARCHITECTURES)
    set(SKIA_TARGET_ARCH "${CMAKE_OSX_ARCHITECTURES}")
else()
    set(SKIA_TARGET_ARCH "${CMAKE_SYSTEM_PROCESSOR}")
endif()
message(STATUS "libskia target architecture: ${SKIA_TARGET_ARCH}")

set(SKIA_IS_ARM64 FALSE)
set(SKIA_IS_X86 FALSE)
if(SKIA_TARGET_ARCH MATCHES "arm64|aarch64")
    set(SKIA_IS_ARM64 TRUE)
elseif(SKIA_TARGET_ARCH MATCHES "x86_64|i386|i686|x64")
    set(SKIA_IS_X86 TRUE)
endif()

# ============================================================
# Skia compile definitions (shared across all targets)
# ============================================================
set(SKIA_DEFINES
    SK_RELEASE
    SK_IGNORE_ETC1_SUPPORT
    SK_SUPPORT_LEGACY_IMAGE_ENCODER_CLASS
    SK_SUPPORT_LEGACY_CANVAS_IS_REFCNT
    SK_SUPPORT_LEGACY_GETTOPDEVICE
    SK_SUPPORT_LEGACY_ACCESSBITMAP
    SK_SUPPORT_LEGACY_CLIP_REGIONOPS
    SK_SUPPORT_LEGACY_GETDEVICE
    SK_SUPPORT_GPU=0
)

# ============================================================
# Skia include directories (shared across all targets)
# ============================================================
set(SKIA_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(SKIA_INCLUDE_DIRS
    ${SKIA_DIR}/include/android
    ${SKIA_DIR}/include/animator
    ${SKIA_DIR}/include/c
    ${SKIA_DIR}/include/codec
    ${SKIA_DIR}/include/config
    ${SKIA_DIR}/include/core
    ${SKIA_DIR}/include/effects
    ${SKIA_DIR}/include/images
    ${SKIA_DIR}/include/pathops
    ${SKIA_DIR}/include/ports
    ${SKIA_DIR}/include/private
    ${SKIA_DIR}/include/svg
    ${SKIA_DIR}/include/utils
    ${SKIA_DIR}/include/utils/ios
    ${SKIA_DIR}/include/utils/mac
    ${SKIA_DIR}/include/utils/win
    ${SKIA_DIR}/include/views
    ${SKIA_DIR}/include/xml
    ${SKIA_DIR}/src/android
    ${SKIA_DIR}/src/animator
    ${SKIA_DIR}/src/c
    ${SKIA_DIR}/src/codec
    ${SKIA_DIR}/src/core
    ${SKIA_DIR}/src/effects
    ${SKIA_DIR}/src/fonts
    ${SKIA_DIR}/src/image
    ${SKIA_DIR}/src/images
    ${SKIA_DIR}/src/lazy
    ${SKIA_DIR}/src/opts
    ${SKIA_DIR}/src/pathops
    ${SKIA_DIR}/src/pdf
    ${SKIA_DIR}/src/pipe
    ${SKIA_DIR}/src/ports
    ${SKIA_DIR}/src/sfnt
    ${SKIA_DIR}/src/svg
    ${SKIA_DIR}/src/utils
    ${SKIA_DIR}/src/views
    ${SKIA_DIR}/src/xml
    ${SKIA_DIR}/src/xps
)

# ============================================================
# Architecture-specific optimization targets
# Each compiled with specific SIMD flags
# ============================================================

# --- opt_none: fallback for unknown architectures ---
add_library(libskia_opt_none STATIC
    src/opts/opts_dummy.cpp
)
# Only add actual none sources on non-x86/non-ARM architectures
if(NOT SKIA_IS_X86 AND NOT SKIA_IS_ARM64)
    target_sources(libskia_opt_none PRIVATE
        src/opts/SkBitmapProcState_opts_none.cpp
        src/opts/SkBlitMask_opts_none.cpp
        src/opts/SkBlitRow_opts_none.cpp
    )
endif()
target_include_directories(libskia_opt_none PRIVATE ${SKIA_INCLUDE_DIRS})
target_compile_definitions(libskia_opt_none PRIVATE ${SKIA_DEFINES})
target_compile_options(libskia_opt_none PRIVATE -w)

# --- opt_arm: ARM/NEON optimizations ---
add_library(libskia_opt_arm STATIC)
target_include_directories(libskia_opt_arm PRIVATE ${SKIA_INCLUDE_DIRS})
target_compile_definitions(libskia_opt_arm PRIVATE ${SKIA_DEFINES})
target_compile_options(libskia_opt_arm PRIVATE -w)

if(SKIA_IS_ARM64)
    # arm64/Apple Silicon: build ARM + NEON + CRC32 opts
    target_sources(libskia_opt_arm PRIVATE
        src/opts/SkBitmapProcState_opts_none.cpp
        src/opts/SkBlitMask_opts_arm.cpp
        src/opts/SkBlitRow_opts_arm.cpp
        src/opts/SkBitmapProcState_arm_neon.cpp
        src/opts/SkBitmapProcState_matrixProcs_neon.cpp
        src/opts/SkBlitMask_opts_arm_neon.cpp
        src/opts/SkBlitRow_opts_arm_neon.cpp
        src/opts/SkOpts_crc32.cpp
    )
elseif(SKIA_TARGET_ARCH MATCHES "armv7")
    target_sources(libskia_opt_arm PRIVATE
        src/opts/SkBitmapProcState_opts_none.cpp
        src/opts/SkBlitMask_opts_arm.cpp
        src/opts/SkBlitRow_opts_arm.cpp
        src/opts/SkBitmapProcState_arm_neon.cpp
        src/opts/SkBitmapProcState_matrixProcs_neon.cpp
        src/opts/SkBlitMask_opts_arm_neon.cpp
        src/opts/SkBlitRow_opts_arm_neon.cpp
        src/opts/SkOpts_crc32.cpp
    )
else()
    # x86/x86_64: just build a stub
    target_sources(libskia_opt_arm PRIVATE
        src/opts/SkOpts_arm_stub.cpp
    )
endif()

# --- Helper macro for x86 SSE/AVX optimization targets ---
# On arm64, these just contain a dummy. On x86_64, they contain real opt code.
macro(add_skia_x86_opt_target TARGET_NAME)
    add_library(${TARGET_NAME} STATIC
        src/opts/opts_dummy.cpp
    )
    target_include_directories(${TARGET_NAME} PRIVATE ${SKIA_INCLUDE_DIRS})
    target_compile_definitions(${TARGET_NAME} PRIVATE ${SKIA_DEFINES})
    target_compile_options(${TARGET_NAME} PRIVATE -w)
endmacro()

# --- opt_sse2 ---
add_skia_x86_opt_target(libskia_opt_sse2)
if(NOT SKIA_IS_ARM64)
    target_sources(libskia_opt_sse2 PRIVATE
        src/opts/SkBitmapProcState_opts_SSE2.cpp
        src/opts/SkBlitRow_opts_SSE2.cpp
        src/opts/opts_check_x86.cpp
    )
    target_compile_options(libskia_opt_sse2 PRIVATE -msse2)
endif()

# --- opt_sse3 (SSSE3) ---
add_skia_x86_opt_target(libskia_opt_sse3)
if(NOT SKIA_IS_ARM64)
    target_sources(libskia_opt_sse3 PRIVATE
        src/opts/SkBitmapProcState_opts_SSSE3.cpp
        src/opts/SkOpts_ssse3.cpp
    )
    target_compile_options(libskia_opt_sse3 PRIVATE -mssse3)
endif()

# --- opt_sse41 ---
add_skia_x86_opt_target(libskia_opt_sse41)
if(NOT SKIA_IS_ARM64)
    target_sources(libskia_opt_sse41 PRIVATE
        src/opts/SkOpts_sse41.cpp
    )
    target_compile_options(libskia_opt_sse41 PRIVATE -msse4.1)
endif()

# --- opt_sse42 ---
add_skia_x86_opt_target(libskia_opt_sse42)
if(NOT SKIA_IS_ARM64)
    target_sources(libskia_opt_sse42 PRIVATE
        src/opts/SkOpts_sse42.cpp
    )
    target_compile_options(libskia_opt_sse42 PRIVATE -msse4.2)
endif()

# --- opt_avx ---
add_skia_x86_opt_target(libskia_opt_avx)
if(NOT SKIA_IS_ARM64)
    target_sources(libskia_opt_avx PRIVATE
        src/opts/SkOpts_avx.cpp
    )
    target_compile_options(libskia_opt_avx PRIVATE -mavx)
endif()

# --- opt_hsw (Haswell: AVX2 + BMI + FMA) ---
add_skia_x86_opt_target(libskia_opt_hsw)
if(NOT SKIA_IS_ARM64)
    target_sources(libskia_opt_hsw PRIVATE
        src/opts/SkOpts_hsw.cpp
    )
    target_compile_options(libskia_opt_hsw PRIVATE -mavx2 -mbmi -mbmi2 -mf16c -mfma)
endif()

# ============================================================
# Main libskia library - all core Skia source files
# ============================================================

# Collect source files by directory (matching GYP's explicit list)
file(GLOB SKIA_CORE_SOURCES      "src/core/*.cpp")
file(GLOB SKIA_EFFECTS_SOURCES   "src/effects/*.cpp")
file(GLOB SKIA_GRADIENT_SOURCES  "src/effects/gradients/*.cpp")
file(GLOB SKIA_FONTS_SOURCES     "src/fonts/*.cpp")
file(GLOB SKIA_IMAGE_SOURCES     "src/image/*.cpp")
file(GLOB SKIA_IMAGES_SOURCES    "src/images/*.cpp")
file(GLOB SKIA_LAZY_SOURCES      "src/lazy/*.cpp")
file(GLOB SKIA_PATHOPS_SOURCES   "src/pathops/*.cpp")
file(GLOB SKIA_PDF_SOURCES       "src/pdf/*.cpp")
file(GLOB SKIA_PIPE_SOURCES      "src/pipe/*.cpp")
file(GLOB SKIA_PORTS_SOURCES     "src/ports/*.cpp")
file(GLOB SKIA_SFNT_SOURCES      "src/sfnt/*.cpp")
file(GLOB SKIA_SVG_SOURCES       "src/svg/*.cpp")
file(GLOB SKIA_UTILS_SOURCES     "src/utils/*.cpp")
file(GLOB SKIA_XML_SOURCES       "src/xml/*.cpp")
file(GLOB SKIA_ANIMATOR_SOURCES  "src/animator/*.cpp")
file(GLOB SKIA_C_SOURCES         "src/c/*.cpp")
file(GLOB SKIA_CODEC_SOURCES     "src/codec/*.cpp")

# Combine all sources
set(SKIA_ALL_SOURCES
    ${SKIA_CORE_SOURCES}
    ${SKIA_EFFECTS_SOURCES}
    ${SKIA_GRADIENT_SOURCES}
    ${SKIA_FONTS_SOURCES}
    ${SKIA_IMAGE_SOURCES}
    ${SKIA_IMAGES_SOURCES}
    ${SKIA_LAZY_SOURCES}
    ${SKIA_PATHOPS_SOURCES}
    ${SKIA_PDF_SOURCES}
    ${SKIA_PIPE_SOURCES}
    ${SKIA_PORTS_SOURCES}
    ${SKIA_SFNT_SOURCES}
    ${SKIA_SVG_SOURCES}
    ${SKIA_UTILS_SOURCES}
    ${SKIA_XML_SOURCES}
    ${SKIA_ANIMATOR_SOURCES}
    ${SKIA_C_SOURCES}
    ${SKIA_CODEC_SOURCES}
)

# Platform-specific util subdirectories
if(APPLE)
    file(GLOB SKIA_UTILS_MAC_SOURCES "src/utils/mac/*.cpp")
    list(APPEND SKIA_ALL_SOURCES ${SKIA_UTILS_MAC_SOURCES})
endif()

# ============================================================
# Source exclusions (matching GYP's sources! and sources/ rules)
# ============================================================

# Always excluded
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkMemory_mozalloc\\.cpp$")
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkLua\\.cpp$")
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkLuaCanvas\\.cpp$")

# Exclude GPU-related files (SK_SUPPORT_GPU=0)
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "[Gg]pu")
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "GPU")

# Exclude XPS (not built on any platform we target)
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "/xps/")

# Codec exclusions: disable most codecs, keep only SkCodec.cpp and SkCodecImageGenerator.cpp
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "KTX")
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "[Ww][Ee][Bb][Pp]")
# Remove all *Codec*.cpp except SkCodec.cpp and SkCodecImageGenerator.cpp
set(SKIA_FILTERED_SOURCES "")
foreach(SRC ${SKIA_ALL_SOURCES})
    get_filename_component(FNAME "${SRC}" NAME)
    if(FNAME MATCHES "Codec" AND FNAME MATCHES "\\.cpp$")
        if(FNAME STREQUAL "SkCodec.cpp" OR FNAME STREQUAL "SkCodecImageGenerator.cpp")
            list(APPEND SKIA_FILTERED_SOURCES "${SRC}")
        endif()
    else()
        list(APPEND SKIA_FILTERED_SOURCES "${SRC}")
    endif()
endforeach()
set(SKIA_ALL_SOURCES ${SKIA_FILTERED_SOURCES})

# Animation scripting exclusions
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkScript")
list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkSnapshot")

# ============================================================
# Platform-specific exclusions
# ============================================================

if(APPLE AND NOT IOS)
    # macOS: exclude Windows-specific, Android-specific, FreeType/FontConfig
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "_win(_dw)?.*\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "/win/")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "WIC\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "_android")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "[Ff]ree[Tt]ype")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "[Ff]ont[Cc]onfig")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "FontConfig")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkFontMgr_custom")

    # Exclude port stubs that have macOS implementations
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkImageEncoder_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkImageGenerator_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkTLS_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkFontHost_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkDebug_android\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkDebug_win\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkOSFile_win\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkOSFile_none\\.cpp$")

elseif(WIN32)
    # Windows: exclude POSIX, FreeType/FontConfig, macOS/CG stuff
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "(pthread|posix)\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "[Ff]ree[Tt]ype")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "FreeType")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "[Ff]ont[Cc]onfig")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "FontConfig")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkFontMgr_custom")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "CG\\.")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "_mac\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "/mac/")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "_android")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkImageEncoder_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkImageGenerator_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkTLS_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkFontMgr_win_gdi_factory\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkDebug_android\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkDebug_stdio\\.cpp$")

elseif(UNIX AND NOT APPLE)
    # Linux: exclude Windows, macOS/CG, Android
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "_win(_dw)?.*\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "/win/")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "WIC\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "CG\\.")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "_mac\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "/mac/")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "_android")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkFontMgr_custom")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkImageEncoder_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkImageGenerator_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkTLS_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkFontHost_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkOSFile_none\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkDebug_android\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkDebug_win\\.cpp$")
    list(FILTER SKIA_ALL_SOURCES EXCLUDE REGEX "SkOSFile_win\\.cpp$")
endif()

# ============================================================
# Create the main library
# ============================================================

add_library(libskia STATIC ${SKIA_ALL_SOURCES})

target_include_directories(libskia PRIVATE ${SKIA_INCLUDE_DIRS})

target_compile_definitions(libskia PRIVATE ${SKIA_DEFINES})

# Silence all warnings for third-party code
target_compile_options(libskia PRIVATE -w)

# ============================================================
# Dependencies
# ============================================================

# Image format libraries (from thirdparty)
if(TARGET libgif)
    target_link_libraries(libskia PRIVATE libgif)
endif()
if(TARGET libjpeg)
    target_link_libraries(libskia PRIVATE libjpeg)
endif()
if(TARGET libpng)
    target_link_libraries(libskia PRIVATE libpng)
endif()
if(TARGET libz)
    target_link_libraries(libskia PRIVATE libz)
elseif(TARGET ZLIB::ZLIB)
    target_link_libraries(libskia PRIVATE ZLIB::ZLIB)
endif()
if(TARGET libexpat)
    target_link_libraries(libskia PRIVATE libexpat)
endif()

# Optimization sub-libraries
target_link_libraries(libskia PRIVATE
    libskia_opt_none
    libskia_opt_arm
    libskia_opt_sse2
    libskia_opt_sse3
    libskia_opt_sse41
    libskia_opt_sse42
    libskia_opt_avx
    libskia_opt_hsw
)

# ============================================================
# Platform-specific settings
# ============================================================

if(APPLE AND NOT IOS)
    # macOS
elseif(UNIX AND NOT APPLE)
    # Linux
    target_compile_definitions(libskia PRIVATE SK_BUILD_FOR_UNIX)
elseif(WIN32)
    target_compile_definitions(libskia PRIVATE
        _WIN7_PLATFORM_UPDATE
        SK_NO_ANALYTIC_AA
    )
endif()

# Freetype dependency for platforms that need it
if(UNIX AND NOT APPLE)
    find_package(Freetype)
    if(FREETYPE_FOUND)
        target_include_directories(libskia PRIVATE ${FREETYPE_INCLUDE_DIRS})
        target_link_libraries(libskia PRIVATE ${FREETYPE_LIBRARIES})
    endif()
endif()

# ============================================================
# Public interface for dependents
# ============================================================

# Propagate defines and include dirs to consumers
target_compile_definitions(libskia PUBLIC
    SK_RELEASE
    SK_SUPPORT_LEGACY_CANVAS_IS_REFCNT
    SK_SUPPORT_LEGACY_GETTOPDEVICE
    SK_SUPPORT_LEGACY_ACCESSBITMAP
    SK_SUPPORT_LEGACY_CLIP_REGIONOPS
    SK_SUPPORT_LEGACY_GETDEVICE
    SK_SUPPORT_GPU=0
)

# Public include dirs (what dependents need to #include Skia headers)
target_include_directories(libskia PUBLIC
    ${SKIA_DIR}/include/config
    ${SKIA_DIR}/include/core
    ${SKIA_DIR}/include/effects
    ${SKIA_DIR}/include/images
    ${SKIA_DIR}/include/pathops
    ${SKIA_DIR}/include/pdf
    ${SKIA_DIR}/include/pipe
    ${SKIA_DIR}/include/ports
    ${SKIA_DIR}/include/svg
    ${SKIA_DIR}/include/utils
    ${SKIA_DIR}/include/private
    # Needed for some legacy methods used by the engine
    ${SKIA_DIR}/src/core
    ${SKIA_DIR}/src/ports
)

if(WIN32)
    target_include_directories(libskia PUBLIC
        ${SKIA_DIR}/src/utils/win
    )
endif()

message(STATUS "libskia: building from source for ${CMAKE_SYSTEM_PROCESSOR}")
