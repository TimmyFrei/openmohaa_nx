cmake_minimum_required(VERSION 3.25)

project(openmohaa_nx LANGUAGES C CXX ASM)

set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)

# ------------------------------------------------------------
# Nintendo Switch target game
#
# 0 = Allied Assault
# 1 = Spearhead
# 2 = Breakthrough
# ------------------------------------------------------------

set(NX_TARGET_GAME 0 CACHE STRING "Nintendo Switch target game (0=AA, 1=Spearhead, 2=Breakthrough)")

if(NOT NX_TARGET_GAME MATCHES "^[012]$")
    message(FATAL_ERROR "NX_TARGET_GAME must be 0, 1 or 2")
endif()

message(STATUS "Nintendo Switch target game: ${NX_TARGET_GAME}")

# ------------------------------------------------------------
# Renderer
# ------------------------------------------------------------

set(NX_RENDERER "gl1" CACHE STRING "Nintendo Switch renderer")
set_property(CACHE NX_RENDERER PROPERTY STRINGS gl1 gl2)

# ------------------------------------------------------------
# Options
# ------------------------------------------------------------

option(NX_ENABLE_NXLINK "Enable nxlink support" OFF)
option(NX_USE_CODEC_MAD "Enable MP3/libmad support" ON)

# ------------------------------------------------------------
# Paths
# ------------------------------------------------------------

set(OPENMOHAA_ROOT "${CMAKE_CURRENT_LIST_DIR}/..")
set(SOURCE_DIR "${OPENMOHAA_ROOT}/code")
set(THIRDPARTY_DIR "${SOURCE_DIR}/thirdparty")

# ------------------------------------------------------------
# Switch definitions
# ------------------------------------------------------------

add_compile_definitions(
    __SWITCH__
    NX_TARGET_GAME=${NX_TARGET_GAME}
)

# ------------------------------------------------------------
# Third-party libraries bundled with OpenMoHAA
# ------------------------------------------------------------

file(GLOB_RECURSE OGG_SOURCES
    "${THIRDPARTY_DIR}/libogg-*/*.c"
)

file(GLOB_RECURSE VORBIS_SOURCES
    "${THIRDPARTY_DIR}/libvorbis-*/*.c"
)

file(GLOB_RECURSE OPUS_SOURCES
    "${THIRDPARTY_DIR}/opus-*/*.c"
)

file(GLOB_RECURSE OPUSFILE_SOURCES
    "${THIRDPARTY_DIR}/opusfile-*/*.c"
)

file(GLOB_RECURSE MAD_SOURCES
    "${THIRDPARTY_DIR}/libmad-*/*.c"
)

# ------------------------------------------------------------
# Include directories
# ------------------------------------------------------------

include_directories(
    "${SOURCE_DIR}"
    "${SOURCE_DIR}/client"
    "${SOURCE_DIR}/server"
    "${SOURCE_DIR}/game"
    "${SOURCE_DIR}/qcommon"
    "${SOURCE_DIR}/script"
    "${SOURCE_DIR}/renderercommon"

    "${THIRDPARTY_DIR}/libogg-1.3.5/include"
    "${THIRDPARTY_DIR}/libvorbis-1.3.7/include"
)

# ------------------------------------------------------------
# Main OpenMoHAA sources
# ------------------------------------------------------------

file(GLOB_RECURSE COMMON_SOURCES
    "${SOURCE_DIR}/qcommon/*.c"
    "${SOURCE_DIR}/qcommon/*.cpp"
    "${SOURCE_DIR}/qcommon/*.h"

    "${SOURCE_DIR}/script/*.c"
    "${SOURCE_DIR}/script/*.cpp"
    "${SOURCE_DIR}/script/*.h"

    "${SOURCE_DIR}/client/*.c"
    "${SOURCE_DIR}/client/*.cpp"
    "${SOURCE_DIR}/client/*.h"

    "${SOURCE_DIR}/renderercommon/*.c"
    "${SOURCE_DIR}/renderercommon/*.cpp"
    "${SOURCE_DIR}/renderercommon/*.h"

    "${SOURCE_DIR}/sys/*.c"
    "${SOURCE_DIR}/sys/*.cpp"
    "${SOURCE_DIR}/sys/*.h"
)

# ------------------------------------------------------------
# Game sources
# ------------------------------------------------------------

file(GLOB_RECURSE GAME_SOURCES
    "${SOURCE_DIR}/fgame/*.c"
    "${SOURCE_DIR}/fgame/*.cpp"
    "${SOURCE_DIR}/fgame/*.h"
)

# ------------------------------------------------------------
# Renderer
# ------------------------------------------------------------

if(NX_RENDERER STREQUAL "gl1")

    file(GLOB_RECURSE RENDERER_SOURCES
        "${SOURCE_DIR}/renderer/*.c"
        "${SOURCE_DIR}/renderer/*.cpp"
        "${SOURCE_DIR}/renderer/*.h"
    )

elseif(NX_RENDERER STREQUAL "gl2")

    file(GLOB_RECURSE RENDERER_SOURCES
        "${SOURCE_DIR}/renderer/*.c"
        "${SOURCE_DIR}/renderer/*.cpp"
        "${SOURCE_DIR}/renderer/*.h"
    )

else()

    message(FATAL_ERROR
        "Unsupported NX_RENDERER: ${NX_RENDERER}"
    )

endif()

# ------------------------------------------------------------
# Platform-specific Switch sources
# ------------------------------------------------------------

file(GLOB_RECURSE NX_PLATFORM_SOURCES
    "${CMAKE_CURRENT_LIST_DIR}/*.c"
    "${CMAKE_CURRENT_LIST_DIR}/*.cpp"
)

# Do not compile CMake files as sources
list(FILTER NX_PLATFORM_SOURCES EXCLUDE REGEX "CMakeLists\\.txt$")

# ------------------------------------------------------------
# SDL / OpenAL / Zlib
#
# Do NOT use find_package(Mad) or find_package(EGL).
# The Switch toolchain does not provide those as desktop-style
# CMake packages.
# ------------------------------------------------------------

find_package(SDL2 REQUIRED)
find_package(OpenAL REQUIRED)
find_package(ZLIB REQUIRED)

# ------------------------------------------------------------
# Executable
# ------------------------------------------------------------

add_executable(openmohaa
    ${COMMON_SOURCES}
    ${GAME_SOURCES}
    ${RENDERER_SOURCES}
    ${NX_PLATFORM_SOURCES}

    ${OGG_SOURCES}
    ${VORBIS_SOURCES}
    ${OPUS_SOURCES}
    ${OPUSFILE_SOURCES}

    ${MAD_SOURCES}
)

# ------------------------------------------------------------
# Include directories from Switch libraries
# ------------------------------------------------------------

target_include_directories(openmohaa PRIVATE

    "${SOURCE_DIR}"

    "${SOURCE_DIR}/client"
    "${SOURCE_DIR}/server"
    "${SOURCE_DIR}/game"
    "${SOURCE_DIR}/qcommon"
    "${SOURCE_DIR}/script"
    "${SOURCE_DIR}/renderercommon"

    "${THIRDPARTY_DIR}/libogg-1.3.5/include"
    "${THIRDPARTY_DIR}/libvorbis-1.3.7/include"

)

# ------------------------------------------------------------
# Compile definitions
# ------------------------------------------------------------

target_compile_definitions(openmohaa PRIVATE
    __SWITCH__
    NX_TARGET_GAME=${NX_TARGET_GAME}
)

if(NX_RENDERER STREQUAL "gl1")
    target_compile_definitions(openmohaa PRIVATE
        USE_RENDERER_GL1
    )
endif()

if(NX_USE_CODEC_MAD)
    target_compile_definitions(openmohaa PRIVATE
        USE_CODEC_MAD
    )
endif()

# ------------------------------------------------------------
# Switch libraries
# ------------------------------------------------------------

target_link_libraries(openmohaa PRIVATE

    SDL2::SDL2
    OpenAL::OpenAL

    ZLIB::ZLIB

    EGL
    glapi

    nx
    drm_nouveau

    m
)

# ------------------------------------------------------------
# nxlink
# ------------------------------------------------------------

if(NX_ENABLE_NXLINK)
    target_compile_definitions(openmohaa PRIVATE
        USE_NXLINK
    )
endif()

# ------------------------------------------------------------
# Switch executable properties
# ------------------------------------------------------------

set_target_properties(openmohaa PROPERTIES
    OUTPUT_NAME "openmohaa"
    SUFFIX ".nro"
)

# ------------------------------------------------------------
# Compiler options
# ------------------------------------------------------------

target_compile_options(openmohaa PRIVATE
    -ffunction-sections
    -fdata-sections
)

target_link_options(openmohaa PRIVATE
    -Wl,--gc-sections
)

# ------------------------------------------------------------
# Status
# ------------------------------------------------------------

message(STATUS "")
message(STATUS "OpenMoHAA Nintendo Switch build")
message(STATUS "  Target game : ${NX_TARGET_GAME}")
message(STATUS "  Renderer    : ${NX_RENDERER}")
message(STATUS "  NXLINK      : ${NX_ENABLE_NXLINK}")
message(STATUS "  MAD         : ${NX_USE_CODEC_MAD}")
message(STATUS "")
