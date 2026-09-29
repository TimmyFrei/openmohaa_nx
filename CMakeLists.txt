cmake_minimum_required(VERSION 3.13)

project(openmohaa_switch VERSION 0.83.0 LANGUAGES C CXX ASM)

if(NOT PLATFORM_SWITCH)
    message(FATAL_ERROR "This CMakeLists.txt is for the Nintendo Switch platform.")
endif()

# -----------------------------------------------------------------------------
# OpenMoHAA source tree
# -----------------------------------------------------------------------------

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../openmohaa/code")
    set(OPENMOHAA_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../openmohaa")
else()
    set(OPENMOHAA_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/..")
endif()

if(NOT EXISTS "${OPENMOHAA_ROOT}/code")
    message(FATAL_ERROR
        "OpenMoHAA source tree not found. "
        "Expected code/ at: ${OPENMOHAA_ROOT}"
    )
endif()

# -----------------------------------------------------------------------------
# Nintendo Switch options
# -----------------------------------------------------------------------------

set(NX_RENDERER "gl1" CACHE STRING
    "Renderer to statically link: gl1 or gl2")
set_property(CACHE NX_RENDERER PROPERTY STRINGS gl1 gl2)

option(NX_ENABLE_NXLINK
    "Redirect stdout/stderr to an nxlink host (debugging)"
    OFF
)

option(NX_USE_CODEC_MAD
    "Enable MP3 playback via libmad"
    ON
)

# -----------------------------------------------------------------------------
# Target game
#
# 0 = Allied Assault
# 1 = Spearhead
# 2 = Breakthrough
# -----------------------------------------------------------------------------

set(NX_TARGET_GAME "0" CACHE STRING
    "OpenMoHAA game: 0=Allied Assault, 1=Spearhead, 2=Breakthrough")

set_property(
    CACHE NX_TARGET_GAME
    PROPERTY STRINGS 0 1 2
)

if(NOT NX_TARGET_GAME MATCHES "^[012]$")
    message(FATAL_ERROR
        "Invalid NX_TARGET_GAME='${NX_TARGET_GAME}'. "
        "Use 0 (Allied Assault), 1 (Spearhead), or 2 (Breakthrough)."
    )
endif()

message(STATUS "Nintendo Switch target game: ${NX_TARGET_GAME}")

# -----------------------------------------------------------------------------
# Global compile definitions
# -----------------------------------------------------------------------------

add_compile_definitions(
    __SWITCH__
    BOTLIB
    APP_MODULE
    _LINUX=1
)

# -----------------------------------------------------------------------------
# Dependencies
# -----------------------------------------------------------------------------

find_package(SDL2 REQUIRED)

find_package(OpenAL REQUIRED)

find_package(Vorbis REQUIRED)
find_package(VorbisFile REQUIRED)
find_package(Ogg REQUIRED)

find_package(Opus REQUIRED)
find_package(OpusFile REQUIRED)

find_package(Mad)

find_package(ZLIB REQUIRED)

find_package(EGL REQUIRED)

# Nintendo Switch GL / Nouveau libraries
find_library(GLAPI_LIBRARY
    NAMES glapi
    REQUIRED
)

find_library(DRM_NOUVEAU_LIBRARY
    NAMES drm_nouveau
    REQUIRED
)

# -----------------------------------------------------------------------------
# Recast / Detour
# -----------------------------------------------------------------------------

add_subdirectory(
    "${OPENMOHAA_ROOT}/code/recastnavigation"
    "${CMAKE_CURRENT_BINARY_DIR}/recastnavigation"
)

# -----------------------------------------------------------------------------
# Common sources
# -----------------------------------------------------------------------------

file(GLOB COMMON_SOURCES
    "${OPENMOHAA_ROOT}/code/qcommon/*.c"
    "${OPENMOHAA_ROOT}/code/qcommon/*.cpp"
)

file(GLOB SCRIPT_SOURCES
    "${OPENMOHAA_ROOT}/code/script/*.c"
    "${OPENMOHAA_ROOT}/code/script/*.cpp"
)

file(GLOB SERVER_SOURCES
    "${OPENMOHAA_ROOT}/code/server/*.c"
    "${OPENMOHAA_ROOT}/code/server/*.cpp"
)

file(GLOB CLIENT_SOURCES
    "${OPENMOHAA_ROOT}/code/client/*.c"
    "${OPENMOHAA_ROOT}/code/client/*.cpp"
)

# -----------------------------------------------------------------------------
# Renderer
# -----------------------------------------------------------------------------

if(NX_RENDERER STREQUAL "gl1")

    file(GLOB RENDERER_SOURCES
        "${OPENMOHAA_ROOT}/code/renderer/*.c"
        "${OPENMOHAA_ROOT}/code/renderer/*.cpp"
    )

    add_compile_definitions(
        USE_RENDERER_GL1
    )

elseif(NX_RENDERER STREQUAL "gl2")

    file(GLOB RENDERER_SOURCES
        "${OPENMOHAA_ROOT}/code/renderer/*.c"
        "${OPENMOHAA_ROOT}/code/renderer/*.cpp"
    )

    add_compile_definitions(
        USE_RENDERER_GL2
    )

else()

    message(FATAL_ERROR
        "Unknown NX_RENDERER='${NX_RENDERER}'. "
        "Use gl1 or gl2."
    )

endif()

# -----------------------------------------------------------------------------
# Nintendo Switch platform sources
# -----------------------------------------------------------------------------

set(SYSTEM_SOURCES
    "${OPENMOHAA_ROOT}/code/sys/sys_main.c"

    "${CMAKE_CURRENT_SOURCE_DIR}/source/nx_app.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/source/nx_compat.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/source/nx_heapguard.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/source/sys_nx.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/source/sys_nx_modules.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/source/sys_nx_new.c"
)

# -----------------------------------------------------------------------------
# Game modules
# -----------------------------------------------------------------------------

file(GLOB GAME_SOURCES
    "${OPENMOHAA_ROOT}/code/game/*.c"
    "${OPENMOHAA_ROOT}/code/game/*.cpp"
)

file(GLOB CGAME_SOURCES
    "${OPENMOHAA_ROOT}/code/cgame/*.c"
    "${OPENMOHAA_ROOT}/code/cgame/*.cpp"
)

# -----------------------------------------------------------------------------
# Include directories
# -----------------------------------------------------------------------------

include_directories(
    "${OPENMOHAA_ROOT}/code"
    "${OPENMOHAA_ROOT}/code/qcommon"
    "${OPENMOHAA_ROOT}/code/client"
    "${OPENMOHAA_ROOT}/code/server"
    "${OPENMOHAA_ROOT}/code/game"
    "${OPENMOHAA_ROOT}/code/cgame"
    "${OPENMOHAA_ROOT}/code/script"
    "${OPENMOHAA_ROOT}/code/renderer"

    "${CMAKE_CURRENT_SOURCE_DIR}/include"

    "${SDL2_INCLUDE_DIRS}"
)

# -----------------------------------------------------------------------------
# OpenMoHAA executable
# -----------------------------------------------------------------------------

set(CLIENT_BINARY openmohaa)

add_executable(
    ${CLIENT_BINARY}

    ${COMMON_SOURCES}
    ${SCRIPT_SOURCES}
    ${SERVER_SOURCES}
    ${CLIENT_SOURCES}
    ${RENDERER_SOURCES}
    ${SYSTEM_SOURCES}
)

# -----------------------------------------------------------------------------
# Game module libraries
# -----------------------------------------------------------------------------

add_library(
    openmohaa_game
    STATIC
    ${GAME_SOURCES}
)

add_library(
    openmohaa_cgame
    STATIC
    ${CGAME_SOURCES}
)

target_link_libraries(
    ${CLIENT_BINARY}
    PRIVATE
    openmohaa_game
    openmohaa_cgame
)

# -----------------------------------------------------------------------------
# Compile definitions
# -----------------------------------------------------------------------------

target_compile_definitions(
    ${CLIENT_BINARY}
    PRIVATE

    USE_OPENAL

    USE_CODEC_VORBIS
    USE_CODEC_OPUS

    USE_INTERNAL_JPEG

    $<$<BOOL:${NX_USE_CODEC_MAD}>:USE_CODEC_MAD>

    $<$<BOOL:${NX_ENABLE_NXLINK}>:NXLINK_ENABLE>

    NX_HEAPGUARD

    NX_TARGET_GAME=${NX_TARGET_GAME}
)

# -----------------------------------------------------------------------------
# Libraries
# -----------------------------------------------------------------------------

target_link_libraries(
    ${CLIENT_BINARY}
    PRIVATE

    ${SDL2_LIBRARIES}

    OpenAL::OpenAL

    Vorbis::vorbis
    Vorbis::vorbisfile
    Ogg::ogg

    Opus::opus
    OpusFile::opusfile

    ZLIB::ZLIB

    ${EGL_LIBRARIES}
    ${GLAPI_LIBRARY}
    ${DRM_NOUVEAU_LIBRARY}

    m
)

if(NX_USE_CODEC_MAD AND Mad_FOUND)
    target_link_libraries(
        ${CLIENT_BINARY}
        PRIVATE
        Mad::mad
    )
endif()

# -----------------------------------------------------------------------------
# Nintendo Switch specific linker settings
# -----------------------------------------------------------------------------

target_link_options(
    ${CLIENT_BINARY}
    PRIVATE

    -specs=nosys.specs
)

# -----------------------------------------------------------------------------
# ROMFS
# -----------------------------------------------------------------------------

set(NX_ROMFS_DIR
    "${CMAKE_CURRENT_SOURCE_DIR}/romfs"
)

# -----------------------------------------------------------------------------
# Nintendo Switch NRO packaging
# -----------------------------------------------------------------------------

if(COMMAND nx_create_nro)

    nx_create_nro(
        ${CLIENT_BINARY}
        VERSION ${PROJECT_VERSION}
        ROMFS ${NX_ROMFS_DIR}
    )

else()

    find_program(
        ELF2NRO
        elf2nro
        REQUIRED
    )

    add_custom_command(
        TARGET ${CLIENT_BINARY}
        POST_BUILD

        COMMAND
            ${ELF2NRO}
            "$<TARGET_FILE:${CLIENT_BINARY}>"
            "${CMAKE_CURRENT_BINARY_DIR}/${CLIENT_BINARY}.nro"

        COMMENT
            "Creating Nintendo Switch NRO"
    )

endif()

# -----------------------------------------------------------------------------
# Build information
# -----------------------------------------------------------------------------

message(STATUS "")
message(STATUS "==============================================")
message(STATUS " OpenMoHAA Nintendo Switch")
message(STATUS "==============================================")
message(STATUS " Renderer:     ${NX_RENDERER}")
message(STATUS " Target game:  ${NX_TARGET_GAME}")
message(STATUS "")
message(STATUS " 0 = Allied Assault")
message(STATUS " 1 = Spearhead")
message(STATUS " 2 = Breakthrough")
message(STATUS "==============================================")
message(STATUS "")
