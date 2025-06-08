cmake_minimum_required(VERSION 3.10...4.0)
project(msdf-atlas-gen VERSION 1.2 LANGUAGES CXX)

set(MAIN_PROJECT OFF)
if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(MAIN_PROJECT ON)
endif()

include(FetchContent)
FetchContent_Declare(
    freetype
    GIT_REPOSITORY https://gitlab.freedesktop.org/freetype/freetype.git
    GIT_TAG        VER-2-13-3
)
FetchContent_MakeAvailable(freetype)

option(MSDF_ATLAS_GEN_BUILD_STANDALONE "Build the msdf-atlas-gen standalone executable" ${MAIN_PROJECT})
option(MSDFGEN_BUILD_MSDFGEN_STANDALONE "Build the msdfgen standalone executable" ${MAIN_PROJECT})
option(MSDFGEN_USE_OPENMP "Build with OpenMP support for multithreaded code" OFF)
option(MSDFGEN_USE_CPP11 "Build with C++11 enabled" ON)
option(MSDFGEN_USE_SKIA "Build with the Skia library" OFF)
option(MSDFGEN_INSTALL "Generate installation target" OFF)
option(FREETYPE_WITH_PNG "Link libpng and zlib because FreeType is configured to require it" OFF)
option(FREETYPE_WITH_HARFBUZZ "Link HarfBuzz because FreeType is configured to require it" OFF)


set(MSDFGEN_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/msdfgen")
project(msdfgen VERSION 1.8 LANGUAGES CXX)

list(APPEND CMAKE_MODULE_PATH "${MSDFGEN_SOURCE_DIR}/cmake")

#----------------------------------------------------------------
# Gathering File
#----------------------------------------------------------------
file(GLOB_RECURSE SOURCES "${MSDFGEN_SOURCE_DIR}/core/*.cpp")
file(GLOB_RECURSE ext_SOURCES "${MSDFGEN_SOURCE_DIR}/ext/*.cpp" "${MSDFGEN_SOURCE_DIR}/lib/*.cpp")

# Build the library (aliased name because it's the same target name the exe)
include(folderize)
folderize_sources(SOURCES ${CMAKE_SOURCE_DIR})
folderize_sources(ext_SOURCES ${CMAKE_SOURCE_DIR})

#----------------------------------------------------------------
# Target configuration
#----------------------------------------------------------------

add_library(msdfgen ${SOURCES})
add_library(msdfgen::msdfgen ALIAS msdfgen)
target_include_directories(msdfgen INTERFACE
	$<INSTALL_INTERFACE:include>
	$<BUILD_INTERFACE:${MSDFGEN_SOURCE_DIR}/>
)

if(MSDFGEN_USE_CPP11)
	target_compile_features(msdfgen PUBLIC cxx_std_11)
	target_compile_definitions(msdfgen PUBLIC MSDFGEN_USE_CPP11)
endif()

add_library(msdfgen-ext ${ext_SOURCES})
add_library(msdfgen::msdfgen-ext ALIAS msdfgen-ext)
target_link_libraries(msdfgen-ext PUBLIC msdfgen::msdfgen freetype)
target_include_directories(msdfgen-ext
PUBLIC
	$<INSTALL_INTERFACE:include>
	$<BUILD_INTERFACE:${MSDFGEN_SOURCE_DIR}/>
	
PRIVATE
	${MSDFGEN_SOURCE_DIR}/include
)

target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_CMAKE_BUILD)

if(MSDFGEN_USE_CPP11)
	target_compile_features(msdfgen-ext PUBLIC cxx_std_11)
	target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_USE_CPP11)
endif()

find_package(Threads REQUIRED)
file(GLOB_RECURSE SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/msdf-atlas-gen/*.cpp")

# msdf-atlas-gen library
add_library(msdf-atlas-gen ${SOURCES})
add_library(msdf-atlas-gen::msdf-atlas-gen ALIAS msdf-atlas-gen)

target_include_directories(msdf-atlas-gen
    INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/msdfgen/include # for lodepng.h
            ${CMAKE_CURRENT_SOURCE_DIR}/artery-font-format
)

if (MSVC)
    target_compile_definitions(msdf-atlas-gen PUBLIC _CRT_SECURE_NO_WARNINGS)
endif()

target_compile_features(msdf-atlas-gen PUBLIC cxx_std_11)
target_link_libraries(msdf-atlas-gen PUBLIC Threads::Threads msdfgen::msdfgen msdfgen::msdfgen-ext)

# TODO make these public in msdfgen so that this doesn't have to be repeated here
if(FREETYPE_WITH_PNG)
    find_package(ZLIB REQUIRED)
    find_package(PNG REQUIRED)
    target_link_libraries(msdf-atlas-gen PUBLIC ZLIB::ZLIB PNG::PNG)
endif()
if(FREETYPE_WITH_HARFBUZZ)
    # No FindHarfBuzz.cmake available so falling back to CMAKE_LIBRARY_PATH
    find_library(FREETYPE_HARFBUZZ_LIB "harfbuzz")
    target_link_libraries(msdf-atlas-gen PUBLIC "${FREETYPE_HARFBUZZ_LIB}")
endif()
