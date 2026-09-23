cmake_minimum_required(VERSION 3.14)
if(POLICY CMP0169)
  cmake_policy(SET CMP0169 OLD)
endif()
project(hunspell C CXX)

set(CMAKE_CXX_STANDARD 14)

# Expect HUNSPELL_VERSION to be passed, e.g. -DHUNSPELL_VERSION=v1.7.2
if(NOT HUNSPELL_VERSION)
    set(HUNSPELL_VERSION "v1.7.2")
endif()

# We can parse major, minor, patch from HUNSPELL_VERSION, assuming it starts with 'v' and format is vX.Y.Z
string(REGEX MATCH "^v?([0-9]+)\\.([0-9]+)\\.([0-9]+)" MATCHED ${HUNSPELL_VERSION})
if(MATCHED)
    set(HUNSPELL_VERSION_MAJOR ${CMAKE_MATCH_1})
    set(HUNSPELL_VERSION_MINOR ${CMAKE_MATCH_2})
    set(HUNSPELL_VERSION_PATCH ${CMAKE_MATCH_3})
    set(VERSION "${CMAKE_MATCH_1}.${CMAKE_MATCH_2}.${CMAKE_MATCH_3}")
else()
    set(HUNSPELL_VERSION_MAJOR 1)
    set(HUNSPELL_VERSION_MINOR 7)
    set(HUNSPELL_VERSION_PATCH 2)
    set(VERSION "1.7.2")
endif()

include(FetchContent)
FetchContent_Declare(
    hunspell_src
    GIT_REPOSITORY https://github.com/hunspell/hunspell.git
    GIT_TAG ${HUNSPELL_VERSION}
)
FetchContent_Populate(hunspell_src)

set(HUNSPELL_SRC_DIR ${hunspell_src_SOURCE_DIR})

file(GLOB_RECURSE SRC_FILES "${HUNSPELL_SRC_DIR}/src/hunspell/*.cxx")
file(GLOB_RECURSE HDR_FILES "${HUNSPELL_SRC_DIR}/src/hunspell/*.hxx" "${HUNSPELL_SRC_DIR}/src/hunspell/*.h")

# Generate headers if they exist in this version
if(EXISTS "${HUNSPELL_SRC_DIR}/src/hunspell/hunversion.h.in")
    configure_file(${HUNSPELL_SRC_DIR}/src/hunspell/hunversion.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/hunspell/hunversion.h @ONLY)
    list(APPEND HDR_FILES "${CMAKE_CURRENT_BINARY_DIR}/include/hunspell/hunversion.h")
endif()

if(EXISTS "${HUNSPELL_SRC_DIR}/src/hunspell/hunvisapi.h.in")
    configure_file(${HUNSPELL_SRC_DIR}/src/hunspell/hunvisapi.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/hunspell/hunvisapi.h @ONLY)
    list(APPEND HDR_FILES "${CMAKE_CURRENT_BINARY_DIR}/include/hunspell/hunvisapi.h")
endif()

# Shared library
add_library(hunspell SHARED ${SRC_FILES})
target_include_directories(hunspell PUBLIC 
    $<BUILD_INTERFACE:${HUNSPELL_SRC_DIR}/src/hunspell>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_compile_definitions(hunspell PRIVATE BUILDING_LIBHUNSPELL)

# Static library
add_library(hunspell_static STATIC ${SRC_FILES})
target_include_directories(hunspell_static PUBLIC 
    $<BUILD_INTERFACE:${HUNSPELL_SRC_DIR}/src/hunspell>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
    $<INSTALL_INTERFACE:include>
)
target_compile_definitions(hunspell_static PUBLIC HUNSPELL_STATIC)
set_target_properties(hunspell_static PROPERTIES OUTPUT_NAME hunspell)

# iOS framework properties
if(IOS)
    set_target_properties(hunspell PROPERTIES
        FRAMEWORK TRUE
        MACOSX_FRAMEWORK_IDENTIFIER org.hunspell.hunspell
        MACOSX_FRAMEWORK_INFO_PLIST ""
        PUBLIC_HEADER "${HDR_FILES}"
    )
endif()

install(TARGETS hunspell hunspell_static
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
        FRAMEWORK DESTINATION lib)

if(NOT IOS)
    install(FILES ${HDR_FILES} DESTINATION include/hunspell)
endif()
