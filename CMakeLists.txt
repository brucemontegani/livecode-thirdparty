# thirdparty - Third-party libraries built from source
# This directory contains vendored third-party libraries

message(STATUS "Configuring third-party libraries")

# Helper function for third-party libraries that silences warnings
function(openxtalk_add_thirdparty_library TARGET_NAME)
    if(TARGET ${TARGET_NAME})
        # Silence warnings for third-party code
        if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
            target_compile_options(${TARGET_NAME} PRIVATE -w)
        elseif(MSVC)
            target_compile_options(${TARGET_NAME} PRIVATE /W0)
        endif()
    endif()
endfunction()

# zlib - compression library
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libz/src")
    add_library(libz STATIC
        libz/src/adler32.c
        libz/src/compress.c
        libz/src/crc32.c
        libz/src/deflate.c
        libz/src/gzclose.c
        libz/src/gzlib.c
        libz/src/gzread.c
        libz/src/gzwrite.c
        libz/src/infback.c
        libz/src/inffast.c
        libz/src/inflate.c
        libz/src/inftrees.c
        libz/src/trees.c
        libz/src/uncompr.c
        libz/src/zutil.c
    )
    target_include_directories(libz PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libz/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libz/include>
    )
    openxtalk_add_thirdparty_library(libz)
    # Only create alias if ZLIB::ZLIB doesn't already exist
    if(NOT TARGET ZLIB::ZLIB)
        add_library(ZLIB::ZLIB ALIAS libz)
    endif()
endif()

# libpng - PNG image library
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libpng/src")
    file(GLOB LIBPNG_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/libpng/src/*.c")
    add_library(libpng STATIC ${LIBPNG_SOURCES})
    target_include_directories(libpng PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libpng/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libpng/include>
    )
    target_link_libraries(libpng PRIVATE libz)
    # Disable ARM NEON optimizations - source files for these are not present
    target_compile_definitions(libpng PRIVATE PNG_ARM_NEON_OPT=0)
    openxtalk_add_thirdparty_library(libpng)
endif()

# libjpeg - JPEG image library
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libjpeg/src")
    file(GLOB LIBJPEG_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/libjpeg/src/*.c")
    add_library(libjpeg STATIC ${LIBJPEG_SOURCES})
    target_include_directories(libjpeg PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libjpeg/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libjpeg/include>
    )
    openxtalk_add_thirdparty_library(libjpeg)
endif()

# libgif - GIF image library
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libgif/src")
    file(GLOB LIBGIF_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/libgif/src/*.c")
    add_library(libgif STATIC ${LIBGIF_SOURCES})
    target_include_directories(libgif PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libgif/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libgif/include>
    )
    openxtalk_add_thirdparty_library(libgif)
endif()

# libffi - Foreign function interface
# libffi is complex to build. Check for prebuilt first, then system, then build from source
set(LIBFFI_FOUND FALSE)

# Check for prebuilt
if(EXISTS "${OPENXTALK_PREBUILT_DIR}/lib/mac/libffi.a" AND APPLE AND NOT IOS)
    # Check architecture compatibility
    execute_process(
        COMMAND lipo -info "${OPENXTALK_PREBUILT_DIR}/lib/mac/libffi.a"
        OUTPUT_VARIABLE LIBFFI_ARCH_INFO
        ERROR_QUIET
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    if(LIBFFI_ARCH_INFO MATCHES "${CMAKE_OSX_ARCHITECTURES}" OR
       (CMAKE_OSX_ARCHITECTURES STREQUAL "x86_64" AND LIBFFI_ARCH_INFO MATCHES "x86_64"))
        add_library(libffi STATIC IMPORTED GLOBAL)
        set_target_properties(libffi PROPERTIES
            IMPORTED_LOCATION "${OPENXTALK_PREBUILT_DIR}/lib/mac/libffi.a"
        )
        target_include_directories(libffi INTERFACE
            "${CMAKE_CURRENT_SOURCE_DIR}/libffi/include_darwin"
        )
        target_compile_definitions(libffi INTERFACE FFI_BUILDING)
        set(LIBFFI_FOUND TRUE)
        message(STATUS "libffi: using prebuilt library")
    endif()
endif()

# If not found, try system libffi
if(NOT LIBFFI_FOUND)
    # On macOS, prefer the system libffi which is properly built for all architectures
    if(APPLE)
        # Use the SDK's libffi
        find_library(SYSTEM_FFI_LIBRARY ffi)
        # The include path needs to be the directory containing ffi.h directly
        # System libffi headers are in .../usr/include/ffi/ffi.h
        find_path(SYSTEM_FFI_INCLUDE_BASE ffi/ffi.h)
        if(SYSTEM_FFI_INCLUDE_BASE)
            set(SYSTEM_FFI_INCLUDE "${SYSTEM_FFI_INCLUDE_BASE}/ffi")
        endif()
        if(SYSTEM_FFI_LIBRARY)
            add_library(libffi INTERFACE)
            target_link_libraries(libffi INTERFACE "${SYSTEM_FFI_LIBRARY}")
            if(SYSTEM_FFI_INCLUDE)
                target_include_directories(libffi INTERFACE "${SYSTEM_FFI_INCLUDE}")
                message(STATUS "libffi include: ${SYSTEM_FFI_INCLUDE}")
            endif()
            target_compile_definitions(libffi INTERFACE FFI_BUILDING)
            set(LIBFFI_FOUND TRUE)
            message(STATUS "libffi: using system library at ${SYSTEM_FFI_LIBRARY}")
        endif()
    else()
        find_library(SYSTEM_FFI_LIBRARY ffi PATHS /opt/homebrew/lib /usr/local/lib /usr/lib)
        find_path(SYSTEM_FFI_INCLUDE ffi.h PATHS /opt/homebrew/include /usr/local/include /usr/include)
        if(SYSTEM_FFI_LIBRARY AND SYSTEM_FFI_INCLUDE)
            add_library(libffi STATIC IMPORTED GLOBAL)
            set_target_properties(libffi PROPERTIES
                IMPORTED_LOCATION "${SYSTEM_FFI_LIBRARY}"
            )
            target_include_directories(libffi INTERFACE "${SYSTEM_FFI_INCLUDE}")
            target_compile_definitions(libffi INTERFACE FFI_BUILDING)
            set(LIBFFI_FOUND TRUE)
            message(STATUS "libffi: using system library at ${SYSTEM_FFI_LIBRARY}")
        endif()
    endif()
endif()

# Last resort: build from source (macOS only for now)
if(NOT LIBFFI_FOUND AND APPLE AND NOT IOS)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libffi/src")
        # Enable assembly language support
        enable_language(ASM)

        # Generic sources
        set(LIBFFI_SOURCES
            libffi/src/closures.c
            libffi/src/debug.c
            libffi/src/java_raw_api.c
            libffi/src/prep_cif.c
            libffi/src/raw_api.c
            libffi/src/types.c
        )

        # Architecture-specific sources
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64|aarch64")
            # arm64 - use iOS sources which support arm64
            set(LIBFFI_ASM_SOURCE "${CMAKE_CURRENT_SOURCE_DIR}/libffi/git_master/darwin_ios/src/aarch64/sysv_arm64.S")
            list(APPEND LIBFFI_SOURCES
                libffi/git_master/darwin_ios/src/aarch64/ffi_arm64.c
            )
            set(LIBFFI_INCLUDE_DIRS
                "${CMAKE_CURRENT_SOURCE_DIR}/libffi/git_master/darwin_ios/include"
                "${CMAKE_CURRENT_SOURCE_DIR}/libffi/git_master/darwin_common/include"
                "${CMAKE_CURRENT_SOURCE_DIR}/libffi/git_master/src"
                "${CMAKE_CURRENT_SOURCE_DIR}/libffi/git_master/src/aarch64"
            )
        else()
            # x86_64
            set(LIBFFI_ASM_SOURCE
                "${CMAKE_CURRENT_SOURCE_DIR}/libffi/src/x86/darwin.S"
                "${CMAKE_CURRENT_SOURCE_DIR}/libffi/src/x86/darwin64.S"
            )
            list(APPEND LIBFFI_SOURCES
                libffi/src/x86/ffi.c
                libffi/src/x86/ffi64.c
            )
            set(LIBFFI_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/libffi/include_darwin")
        endif()

        add_library(libffi STATIC ${LIBFFI_SOURCES} ${LIBFFI_ASM_SOURCE})

        # Set assembly file properties to ensure proper compilation
        set_source_files_properties(${LIBFFI_ASM_SOURCE} PROPERTIES
            LANGUAGE ASM
            COMPILE_FLAGS "-x assembler-with-cpp"
        )

        target_include_directories(libffi PUBLIC ${LIBFFI_INCLUDE_DIRS})
        target_compile_definitions(libffi PUBLIC FFI_BUILDING)
        openxtalk_add_thirdparty_library(libffi)
        set(LIBFFI_FOUND TRUE)
        message(STATUS "libffi: building from source for ${CMAKE_SYSTEM_PROCESSOR}")
    endif()
endif()

if(NOT LIBFFI_FOUND)
    message(WARNING "libffi: not found, some features may be unavailable")
endif()

# libpcre - Regular expressions
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libpcre/src")
    file(GLOB LIBPCRE_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/libpcre/src/*.c")
    add_library(libpcre STATIC ${LIBPCRE_SOURCES})
    target_include_directories(libpcre PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libpcre/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libpcre/include>
    )
    target_compile_definitions(libpcre PRIVATE HAVE_CONFIG_H)
    openxtalk_add_thirdparty_library(libpcre)
endif()

# libsqlite - SQLite database
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libsqlite/src")
    add_library(libsqlite STATIC
        libsqlite/src/sqlite3.c
    )
    target_include_directories(libsqlite PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libsqlite/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libsqlite/include>
    )
    target_compile_definitions(libsqlite PRIVATE
        SQLITE_ENABLE_FTS3
        SQLITE_ENABLE_RTREE
    )
    openxtalk_add_thirdparty_library(libsqlite)
endif()

# libxml - XML parsing (libxml2)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libxml/src")
    file(GLOB LIBXML_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/libxml/src/*.c")
    # Exclude test programs, xmllint, xmlcatalog, and trio files
    list(FILTER LIBXML_SOURCES EXCLUDE REGEX "(test|run|xmllint|xmlcatalog|trio)[^/]*\\.c$")
    add_library(libxml STATIC ${LIBXML_SOURCES})
    target_include_directories(libxml
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libxml/include>
        PRIVATE
            ${CMAKE_CURRENT_SOURCE_DIR}/libxml/src
    )
    target_compile_definitions(libxml
        PRIVATE
            LIBXML_STATIC_FOR_DLL=1
        PUBLIC
            LIBXML_STATIC=1
    )
    target_link_libraries(libxml PRIVATE libz)
    openxtalk_add_thirdparty_library(libxml)
endif()

# libxslt - XSLT transformation
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libxslt/src")
    file(GLOB LIBXSLT_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/libxslt/src/*.c")
    add_library(libxslt STATIC ${LIBXSLT_SOURCES})
    target_include_directories(libxslt
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libxslt/include>
        PRIVATE
            ${CMAKE_CURRENT_SOURCE_DIR}/libxslt/include/libxslt
            ${CMAKE_CURRENT_SOURCE_DIR}/libxslt/src
    )
    target_link_libraries(libxslt PUBLIC libxml)
    openxtalk_add_thirdparty_library(libxslt)
endif()

# libzip - ZIP archive support
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libzip/src")
    file(GLOB LIBZIP_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/libzip/src/*.c")
    add_library(libzip STATIC ${LIBZIP_SOURCES})
    target_include_directories(libzip PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libzip/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libzip/include>
    )
    target_link_libraries(libzip PRIVATE libz)
    openxtalk_add_thirdparty_library(libzip)
endif()

# libexpat - XML parsing library (needed by Skia)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libexpat/lib")
    add_library(libexpat STATIC
        libexpat/lib/xmlparse.c
        libexpat/lib/xmlrole.c
        libexpat/lib/xmltok.c
        libexpat/lib/xmltok_impl.c
        libexpat/lib/xmltok_ns.c
    )
    target_include_directories(libexpat PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libexpat>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/libexpat/lib>
    )
    target_compile_definitions(libexpat PRIVATE HAVE_EXPAT_CONFIG_H)
    openxtalk_add_thirdparty_library(libexpat)
endif()

# libskia - 2D graphics library (built from source)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libskia/src")
    add_subdirectory(libskia)
endif()

# libcairo - 2D graphics (complex build, typically prebuilt)
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libcairo")
    message(STATUS "libcairo: expected to use prebuilt library")
endif()

# libfreetype - Font rendering
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libfreetype")
    message(STATUS "libfreetype: expected to use prebuilt library")
endif()

# libharfbuzz - Text shaping
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libharfbuzz")
    message(STATUS "libharfbuzz: expected to use prebuilt library")
endif()

message(STATUS "Third-party libraries configured")
