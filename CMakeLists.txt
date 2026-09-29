cmake_minimum_required(VERSION 3.25)

project(openmohaa_nx LANGUAGES C CXX ASM)

set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)

set(NX_TARGET_GAME 0 CACHE STRING "Nintendo Switch target game (0=AA, 1=Spearhead, 2=Breakthrough)")
set(NX_RENDERER "gl1" CACHE STRING "Nintendo Switch renderer")

option(NX_ENABLE_NXLINK "Enable nxlink support" OFF)
option(NX_USE_CODEC_MAD "Enable MP3/libmad support" ON)

if(NOT NX_TARGET_GAME MATCHES "^[012]$")
    message(FATAL_ERROR "NX_TARGET_GAME must be 0, 1 or 2")
endif()

message(STATUS "Nintendo Switch target game: ${NX_TARGET_GAME}")

set(OPENMOHAA_ROOT "${CMAKE_CURRENT_LIST_DIR}/..")
set(SOURCE_DIR "${OPENMOHAA_ROOT}/code")
set(THIRDPARTY_DIR "${SOURCE_DIR}/thirdparty")

add_compile_definitions(
    __SWITCH__
    NX_TARGET_GAME=${NX_TARGET_GAME}
)

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

file(GLOB_RECURSE COMMON_SOURCES
    "${SOURCE_DIR}/qcommon/*.c"
    "${SOURCE_DIR}/qcommon/*.cpp"
    "${SOURCE_DIR}/script/*.c"
    "${SOURCE_DIR}/script/*.cpp"
    "${SOURCE_DIR}/client/*.c"
    "${SOURCE_DIR}/client/*.cpp"
    "${SOURCE_DIR}/renderercommon/*.c"
    "${SOURCE_DIR}/renderercommon/*.cpp"
    "${SOURCE_DIR}/sys/*.c"
    "${SOURCE_DIR}/sys/*.cpp"
)

file(GLOB_RECURSE GAME_SOURCES
    "${SOURCE_DIR}/fgame/*.c"
    "${SOURCE_DIR}/fgame/*.cpp"
)

file(GLOB_RECURSE RENDERER_SOURCES
    "${SOURCE_DIR}/renderer/*.c"
    "${SOURCE_DIR}/renderer/*.cpp"
)

file(GLOB_RECURSE NX_PLATFORM_SOURCES
    "${CMAKE_CURRENT_LIST_DIR}/*.c"
    "${CMAKE_CURRENT_LIST_DIR}/*.cpp"
)

find_package(SDL2 REQUIRED)
find_package(OpenAL REQUIRED)
find_package(ZLIB REQUIRED)

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

target_link_libraries(openmohaa PRIVATE
    SDL2::SDL2
    OpenAL::OpenAL
    ZLIB::ZLIB
    glapi
    nx
    drm_nouveau
    m
)

if(NX_ENABLE_NXLINK)
    target_compile_definitions(openmohaa PRIVATE
        USE_NXLINK
    )
endif()

set_target_properties(openmohaa PROPERTIES
    OUTPUT_NAME "openmohaa"
    SUFFIX ".nro"
)

target_compile_options(openmohaa PRIVATE
    -ffunction-sections
    -fdata-sections
)

target_link_options(openmohaa PRIVATE
    -Wl,--gc-sections
)

message(STATUS "")
message(STATUS "OpenMoHAA Nintendo Switch build")
message(STATUS "  Target game : ${NX_TARGET_GAME}")
message(STATUS "  Renderer    : ${NX_RENDERER}")
message(STATUS "  NXLINK      : ${NX_ENABLE_NXLINK}")
message(STATUS "  MAD         : ${NX_USE_CODEC_MAD}")
message(STATUS "")
