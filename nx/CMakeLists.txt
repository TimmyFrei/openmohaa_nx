cmake_minimum_required(VERSION 3.18)

project(openmohaa_switch VERSION 0.83.0 LANGUAGES C CXX ASM)

if(NOT PLATFORM_SWITCH)
    message(FATAL_ERROR
        "This project must be configured with the Switch toolchain:\n"
        " -DCMAKE_TOOLCHAIN_FILE=cmake/toolchain/devkita64-libnx.cmake")
endif()

# =============================================================================
# Paths & options
# =============================================================================

set(OPENMOHAA_ROOT "${_NX_DEFAULT_ROOT}" CACHE PATH
    "Path to the upstream OpenMoHAA checkout")

if(NOT EXISTS "${OPENMOHAA_ROOT}/code")
    set(OPENMOHAA_ROOT "${CMAKE_CURRENT_LIST_DIR}/.." CACHE PATH
        "Path to the upstream OpenMoHAA checkout" FORCE)
endif()

set(SRC "${OPENMOHAA_ROOT}/code")
set(NXSRC "${CMAKE_CURRENT_SOURCE_DIR}/source")
set(NXINC "${CMAKE_CURRENT_SOURCE_DIR}/include")

set(NX_RENDERER "gl1" CACHE STRING
    "Renderer to statically link: gl1 or gl2")

set_property(CACHE NX_RENDERER PROPERTY STRINGS gl1 gl2)

# Nintendo Switch target game:
#   0 = Allied Assault
#   1 = Spearhead
#   2 = Breakthrough
set(NX_TARGET_GAME 0 CACHE STRING
    "Nintendo Switch target game (0=Allied Assault, 1=Spearhead, 2=Breakthrough)")

if(NOT NX_TARGET_GAME MATCHES "^[012]$")
    message(FATAL_ERROR
        "NX_TARGET_GAME must be 0, 1 or 2")
endif()

message(STATUS
    "Nintendo Switch target game: ${NX_TARGET_GAME}")

option(NX_ENABLE_NXLINK
    "Enable nxlink support" OFF)

option(NX_USE_CODEC_MAD
    "Enable MP3/libmad support" ON)

option(NX_HEAPGUARD
    "Enable Switch heap guard" OFF)

# =============================================================================
# Compiler configuration
# =============================================================================

set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# The Switch platform header supplies the operating-system definitions
# expected by the upstream OpenMoHAA sources.
add_compile_options(
    -include "${NXINC}/nx_platform.h"
    -idirafter "${CMAKE_CURRENT_SOURCE_DIR}/compat"
)

add_compile_definitions(
    __SWITCH__
    BOTLIB
    APP_MODULE
    _LINUX=1
    NX_TARGET_GAME=${NX_TARGET_GAME}
)

# =============================================================================
# Include directories
# =============================================================================

include_directories(
    ${SRC}/qcommon
    ${SRC}/script
    ${SRC}/sys
    ${SRC}/gamespy/common
    ${SRC}/gamespy/gcdkey
    ${NXINC}
)

include_directories(
    SYSTEM
    "$ENV{DEVKITPRO}/portlibs/switch/include"
)

# =============================================================================
# Switch libraries
# =============================================================================

find_library(NX_LIB_OPENAL
    openal
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_VORBIS
    vorbis
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_VORBISF
    vorbisfile
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_OGG
    ogg
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_OPUS
    opus
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_OPUSFILE
    opusfile
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_MAD
    mad
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_Z
    z
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_EGL
    EGL
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_GLAPI
    glapi
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

find_library(NX_LIB_DRM_NOUV
    drm_nouveau
    PATHS "$ENV{DEVKITPRO}/portlibs/switch/lib"
    NO_DEFAULT_PATH
)

foreach(_lib
    NX_LIB_OPENAL
    NX_LIB_VORBIS
    NX_LIB_VORBISF
    NX_LIB_OGG
    NX_LIB_OPUS
    NX_LIB_OPUSFILE
    NX_LIB_Z
    NX_LIB_EGL
    NX_LIB_GLAPI
    NX_LIB_DRM_NOUV
)
    if(NOT ${_lib})
        message(FATAL_ERROR
            "Required Switch library not found: ${_lib}")
    endif()
endforeach()

if(NX_USE_CODEC_MAD AND NOT NX_LIB_MAD)
    message(WARNING
        "libmad was requested but could not be found. MP3 support may be unavailable.")
endif()

# =============================================================================
# SDL2
# =============================================================================

find_package(SDL2 REQUIRED)

# =============================================================================
# Recast / Detour
# =============================================================================

set(RECAST_ROOT
    "${SRC}/thirdparty/recastnavigation"
)

file(GLOB RECAST_SOURCES
    "${RECAST_ROOT}/Recast/Source/*.cpp"
)

file(GLOB DETOUR_SOURCES
    "${RECAST_ROOT}/Detour/Source/*.cpp"
)

file(GLOB DETOUR_CROWD_SOURCES
    "${RECAST_ROOT}/DetourCrowd/Source/*.cpp"
)

add_library(Recast STATIC
    ${RECAST_SOURCES}
)

target_include_directories(Recast PUBLIC
    "${RECAST_ROOT}/Recast/Include"
)

add_library(Detour STATIC
    ${DETOUR_SOURCES}
)

target_include_directories(Detour PUBLIC
    "${RECAST_ROOT}/Detour/Include"
)

add_library(DetourCrowd STATIC
    ${DETOUR_CROWD_SOURCES}
)

target_include_directories(DetourCrowd PUBLIC
    "${RECAST_ROOT}/DetourCrowd/Include"
    "${RECAST_ROOT}/Detour/Include"
)

# =============================================================================
# JPEG
# =============================================================================

file(GLOB JPEG_SOURCES
    "${SRC}/thirdparty/jpeg-9f/j*.c"
)

set_source_files_properties(
    ${JPEG_SOURCES}
    PROPERTIES
    COMPILE_OPTIONS "-w"
)

# =============================================================================
# GameSpy
# =============================================================================

set(GAMESPY_COMMON_SOURCES
    ${SRC}/gamespy/common/gamespy_common.cpp
)

set(GAMESPY_GCDKEY_SOURCES
    ${SRC}/gamespy/gcdkey/gcdkey.cpp
)

add_library(gcd_common STATIC
    ${GAMESPY_COMMON_SOURCES}
)

target_include_directories(gcd_common PUBLIC
    ${SRC}/gamespy/common
)

add_library(gcd_key STATIC
    ${GAMESPY_GCDKEY_SOURCES}
)

target_include_directories(gcd_key PUBLIC
    ${SRC}/gamespy/gcdkey
    ${SRC}/gamespy/common
)

# =============================================================================
# Common engine sources
# =============================================================================

set(COMMON_SOURCES

    ${SRC}/qcommon/cm_trace.cpp
    ${SRC}/qcommon/cm_world.cpp
    ${SRC}/qcommon/common.cpp
    ${SRC}/qcommon/cvar.cpp
    ${SRC}/qcommon/files.cpp
    ${SRC}/qcommon/md4.cpp
    ${SRC}/qcommon/msg.cpp
    ${SRC}/qcommon/net_chan.cpp
    ${SRC}/qcommon/q_math.cpp
    ${SRC}/qcommon/q_shared.cpp
    ${SRC}/qcommon/qcommon.cpp
    ${SRC}/qcommon/sv_ccmds.cpp
    ${SRC}/qcommon/sv_client.cpp
    ${SRC}/qcommon/sv_game.cpp
    ${SRC}/qcommon/sv_init.cpp
    ${SRC}/qcommon/sv_main.cpp
    ${SRC}/qcommon/sv_net_chan.cpp
    ${SRC}/qcommon/sv_snapshot.cpp
    ${SRC}/qcommon/sv_world.cpp
    ${SRC}/qcommon/usercmd.cpp

    ${SRC}/script/script.cpp
    ${SRC}/script/script_variable.cpp
    ${SRC}/script/script_token.cpp
    ${SRC}/script/script_thread.cpp
    ${SRC}/script/script_threadlist.cpp
    ${SRC}/script/script_class.cpp
    ${SRC}/script/script_struct.cpp
    ${SRC}/script/script_exception.cpp
    ${SRC}/script/script_error.cpp

    ${SRC}/sys/con_log.c
    ${SRC}/sys/con_passive.c
    ${SRC}/sys/sys_main.c
    ${SRC}/sys/win_bounds.cpp
    ${SRC}/sys/win_localization.cpp
    ${SRC}/sys/sys_update_checker.cpp
)

# =============================================================================
# Client sources
# =============================================================================

file(GLOB CLIENT_SOURCES
    "${SRC}/client/*.cpp"
    "${SRC}/client/*.c"
)

# =============================================================================
# Renderer
# =============================================================================

file(GLOB RENDERER_COMMON_SOURCES
    "${SRC}/renderercommon/*.cpp"
    "${SRC}/renderercommon/*.c"
)

if(NX_RENDERER STREQUAL "gl1")

    file(GLOB RENDERER_BACKEND_SOURCES
        "${SRC}/renderer/*.cpp"
        "${SRC}/renderer/*.c"
    )

    add_compile_definitions(
        USE_RENDERER_GL1
    )

elseif(NX_RENDERER STREQUAL "gl2")

    file(GLOB RENDERER_BACKEND_SOURCES
        "${SRC}/renderer/*.cpp"
        "${SRC}/renderer/*.c"
    )

    add_compile_definitions(
        USE_RENDERER_GL2
    )

else()

    message(FATAL_ERROR
        "Unsupported NX_RENDERER: ${NX_RENDERER}")

endif()

set(RENDERER_SOURCES
    ${RENDERER_COMMON_SOURCES}
    ${RENDERER_BACKEND_SOURCES}
    ${JPEG_SOURCES}
)

# =============================================================================
# Switch system sources
# =============================================================================

set(SYSTEM_SOURCES

    ${SRC}/sys/con_log.c
    ${SRC}/sys/con_passive.c
    ${SRC}/sys/sys_main.c
    ${SRC}/sys/win_bounds.cpp
    ${SRC}/sys/win_localization.cpp

    ${NXSRC}/nx_app.c
    ${NXSRC}/sys_nx.c
    ${NXSRC}/sys_nx_new.c
    ${NXSRC}/sys_nx_modules.c
    ${NXSRC}/nx_compat.c
    ${NXSRC}/nx_heapguard.cpp

    ${SRC}/sys/sys_update_checker.cpp
)

# =============================================================================
# Static game / cgame modules
# =============================================================================

file(GLOB GAME_SOURCES
    "${SRC}/fgame/*.cpp"
    "${SRC}/fgame/*.c"
)

file(GLOB CGAME_SOURCES
    "${SRC}/cgame/*.cpp"
    "${SRC}/cgame/*.c"
)

add_library(game_static STATIC
    ${GAME_SOURCES}
)

target_include_directories(game_static PRIVATE
    ${SRC}
    ${SRC}/qcommon
    ${SRC}/fgame
    ${SRC}/script
)

target_compile_definitions(game_static PRIVATE
    __SWITCH__
    BOTLIB
    APP_MODULE
    _LINUX=1
    NX_TARGET_GAME=${NX_TARGET_GAME}
)

add_library(cgame_static STATIC
    ${CGAME_SOURCES}
)

target_include_directories(cgame_static PRIVATE
    ${SRC}
    ${SRC}/qcommon
    ${SRC}/cgame
    ${SRC}/script
)

target_compile_definitions(cgame_static PRIVATE
    __SWITCH__
    BOTLIB
    APP_MODULE
    _LINUX=1
    NX_TARGET_GAME=${NX_TARGET_GAME}
)

# =============================================================================
# Main executable
# =============================================================================

add_executable(openmohaa

    ${COMMON_SOURCES}
    ${CLIENT_SOURCES}
    ${SYSTEM_SOURCES}
    ${RENDERER_SOURCES}
)

target_include_directories(openmohaa PRIVATE

    ${SRC}
    ${SRC}/qcommon
    ${SRC}/script
    ${SRC}/sys
    ${SRC}/client
    ${SRC}/renderer
    ${SRC}/renderercommon

    ${SRC}/gamespy/common
    ${SRC}/gamespy/gcdkey

    ${NXSRC}
    ${NXINC}

    "$ENV{DEVKITPRO}/portlibs/switch/include"
)

# =============================================================================
# Compile definitions
# =============================================================================

target_compile_definitions(openmohaa PRIVATE

    __SWITCH__
    BOTLIB
    APP_MODULE
    _LINUX=1

    NX_TARGET_GAME=${NX_TARGET_GAME}
)

if(NX_USE_CODEC_MAD)
    target_compile_definitions(openmohaa PRIVATE
        USE_CODEC_MAD
    )
endif()

if(NX_ENABLE_NXLINK)
    target_compile_definitions(openmohaa PRIVATE
        USE_NXLINK
    )
endif()

if(NX_HEAPGUARD)
    target_compile_definitions(openmohaa PRIVATE
        NX_HEAPGUARD
    )
endif()

# =============================================================================
# Link
# =============================================================================

target_link_libraries(openmohaa PRIVATE

    -Wl,--start-group

    gcd_key
    gcd_common

    Detour
    DetourCrowd
    Recast

    game_static
    cgame_static

    -Wl,--end-group

    SDL2::SDL2

    ${NX_LIB_OPENAL}

    ${NX_LIB_VORBISF}
    ${NX_LIB_VORBIS}
    ${NX_LIB_OGG}

    ${NX_LIB_OPUSFILE}
    ${NX_LIB_OPUS}

    ${NX_LIB_Z}

    ${NX_LIB_EGL}
    ${NX_LIB_GLAPI}
    ${NX_LIB_DRM_NOUVEAU}

    nx
    m
)

if(NX_USE_CODEC_MAD AND NX_LIB_MAD)
    target_link_libraries(openmohaa PRIVATE
        ${NX_LIB_MAD}
    )
endif()

# =============================================================================
# Compiler / linker flags
# =============================================================================

target_compile_options(openmohaa PRIVATE
    -ffunction-sections
    -fdata-sections
)

target_link_options(openmohaa PRIVATE
    -Wl,--gc-sections
)

# =============================================================================
# NRO packaging
# =============================================================================

set(OPENMOHAA_ELF
    "${CMAKE_CURRENT_BINARY_DIR}/openmohaa.elf"
)

set(OPENMOHAA_NRO
    "${CMAKE_CURRENT_BINARY_DIR}/openmohaa.nro"
)

set_target_properties(openmohaa PROPERTIES
    OUTPUT_NAME "openmohaa"
)

add_custom_command(
    TARGET openmohaa
    POST_BUILD

    COMMAND
        ${CMAKE_OBJCOPY}
        -O binary
        "$<TARGET_FILE:openmohaa>"
        "${OPENMOHAA_ELF}"

    COMMAND
        elf2nro
        "${OPENMOHAA_ELF}"
        "${OPENMOHAA_NRO}"

    BYPRODUCTS
        "${OPENMOHAA_NRO}"

    COMMENT
        "Packaging openmohaa.elf into openmohaa.nro"
)

# =============================================================================
# Status
# =============================================================================

message(STATUS "")
message(STATUS "OpenMoHAA Nintendo Switch build")
message(STATUS "  Target game : ${NX_TARGET_GAME}")
message(STATUS "  Renderer    : ${NX_RENDERER}")
message(STATUS "  NXLINK      : ${NX_ENABLE_NXLINK}")
message(STATUS "  MAD         : ${NX_USE_CODEC_MAD}")
message(STATUS "  Heap guard  : ${NX_HEAPGUARD}")
message(STATUS "")
