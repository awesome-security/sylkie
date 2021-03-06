# Copyright (c) 2017 Daniel L. Robertson
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.7)
project(
    sylkie
    VERSION 0.0.1
    LANGUAGES C
    )

set(LIB_DIR "${PROJECT_SOURCE_DIR}/lib")
set(EXE_DIR "${PROJECT_SOURCE_DIR}/src")
set(INCLUDE_DIR "${PROJECT_SOURCE_DIR}/include")
set(CMAKE_EXTRA "${PROJECT_SOURCE_DIR}/CMake")

include(CheckIncludeFile)

# Check OS

message("-- Check for OS support")

if(NOT UNIX)
    message(FATAL_ERROR "This project is not yet supported on non-unix systems")
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(BUILD_OS "linux")
    SET(BUILD_LINUX 1)
else(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    message(FATAL_ERROR "This project is not yet supported on ${CMAKE_SYSTEM_NAME}")
endif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")

# Check kernel version

message("-- Check kernel version")

set(KERNEL_VERSION_STRING ${CMAKE_SYSTEM_VERSION})

string(REGEX MATCH "([0-9]+)\\.([0-9]+)[.-]([0-9]+)"
    KERNEL_MATCHED "${KERNEL_VERSION_STRING}")

if(NOT KERNEL_MATCHED)
    message(FATAL_ERROR "${CMAKE_SYSTEM_VERSION} did not match kernel regex")
endif(NOT KERNEL_MATCHED)

set(KERNEL_MAJOR_VERSION ${CMAKE_MATCH_1})
set(KERNEL_MINOR_VERSION ${CMAKE_MATCH_2})
set(KERNEL_PATCH_VERSION ${CMAKE_MATCH_3})

set(KERNEL_VERSION "${CMAKE_MATCH_1}.${CMAKE_MATCH_2}.${CMAKE_MATCH_4}")

if(KERNEL_VERSION VERSION_LESS 3.0)
    message(FATAL_ERROR "Bad kernel version")
else(KERNEL_VERSION VERSION_LESS 3.0)
    message("-- Check kernel version -- works")
endif(KERNEL_VERSION VERSION_LESS 3.0)

# Check for headers in CMake/headers.txt

function(header_check filename)
    if(EXISTS ${filename})
        file(STRINGS ${filename} HEADERS REGEX "[A-z\/]+\\.h")
        foreach(f ${HEADERS})
            check_include_file(${f} HAVE_HEADER_${f})
            if(NOT HAVE_HEADER_${f})
                message(FATAL_ERROR "Could not find ${f}")
            endif(NOT HAVE_HEADER_${f})
        endforeach(f)
    endif(EXISTS ${filename})
endfunction(header_check)

# Check for default headers
header_check(${CMAKE_EXTRA}/headers.txt)

# Check for OS specific headers
header_check(${CMAKE_EXTRA}/${BUILD_OS}/headers.txt)

# Basic build

include_directories(${INCLUDE_DIR})

file(GLOB INCLUDE_FILES ${INCLUDE_DIR}/*.h)

file(GLOB LIB_SRCS ${LIB_DIR}/*.c)

file(GLOB EXE_SRCS ${EXE_DIR}/*.c)

# Configure config.h

configure_file(${INCLUDE_DIR}/config.in ${INCLUDE_DIR}/config.h)

# CFLAGS

set(DEFAULT_DEBUG "-std=c11 -g -ggdb -O0")
set(DEV_WARNINGS "-Wall -Werror -Wshadow -Wstrict-overflow -fno-strict-aliasing")
set(CMAKE_C_FLAGS_DEBUG
    "${CMAKE_C_FLAGS_DEBUG} ${DEFAULT_DEBUG} ${DEV_WARNINGS}")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -std=c11 -O2 -Wall")

# Add custom build type

# Add new build types
set(CMAKE_C_FLAGS_ASAN
    "${DEFAULT_DEBUG} ${DEV_WARNINGS} -fsanitize=address -fno-omit-frame-pointer"
    CACHE STRING "Flags used by the C++ compiler during address sanitizer builds."
    FORCE )
set(CMAKE_EXE_LINKER_FLAGS_ASAN
    ""
    CACHE STRING "Flags used for linking binaries during address sanitizer builds."
    FORCE )
set(CMAKE_SHARED_LINKER_FLAGS_ASAN
    ""
    CACHE STRING "Flags used by the shared libraries linker during address
    sanitizer builds."
    FORCE )
mark_as_advanced(
    CMAKE_CXX_FLAGS_ASAN
    CMAKE_C_FLAGS_ASAN
    CMAKE_EXE_LINKER_FLAGS_ASAN
    CMAKE_SHARED_LINKER_FLAGS_ASAN
    )

# Outputs

add_executable(${PROJECT_NAME}-bin ${EXE_SRCS} ${LIB_SRCS})
set_target_properties(${PROJECT_NAME}-bin PROPERTIES
    OUTPUT_NAME ${PROJECT_NAME})
target_include_directories(${PROJECT_NAME}-bin PRIVATE ${EXE_DIR}/include)
target_link_libraries(${PROJECT_NAME}-bin PRIVATE json-c)
target_link_libraries(${PROJECT_NAME}-bin PRIVATE seccomp)

add_library(${PROJECT_NAME}-shared SHARED ${LIB_SRCS})
set_target_properties(${PROJECT_NAME}-shared PROPERTIES
    OUTPUT_NAME ${PROJECT_NAME})
set_target_properties(${PROJECT_NAME}-shared PROPERTIES
    SOVERSION ${PROJECT_VERSION_MAJOR})

add_library(${PROJECT_NAME}-static STATIC ${LIB_SRCS})
set_target_properties(${PROJECT_NAME}-static PROPERTIES
    OUTPUT_NAME ${PROJECT_NAME})

# Lints

file(GLOB_RECURSE ALL_SOURCE_FILES ${SRC_DIR}/*.c ${INCLUDE_DIR}/*.h)

add_custom_target(clang-tidy COMMAND clang-tidy ${EXE_DIR}/*.c ${LIB_DIR}/*.c
    -- -std=c11 -I${INCLUDE_DIR} -I${EXE_DIR}/include)

add_custom_target(clang-format COMMAND clang-format -i -style=file
    ${EXE_DIR}/*.c ${EXE_DIR}/include/*.h ${LIB_DIR}/*.c ${INCLUDE_DIR}/*.h)

# Docs

find_package(Doxygen)
if(DOXYGEN_FOUND)
    set(DOC_FILES "")
    foreach(str ${LIB_SRCS})
        set(DOC_FILES "${DOC_FILES} ${str}")
    endforeach(str)
    foreach(str ${INCLUDE_FILES})
        set(DOC_FILES "${DOC_FILES} ${str}")
    endforeach(str)
    add_custom_target(doc
        ${DOXYGEN_EXECUTABLE}
        ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
        WORKING DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        )
endif(DOXYGEN_FOUND)

# Install

install(
    TARGETS
    ${PROJECT_NAME}-bin
    ${PROJECT_NAME}-shared
    ${PROJECT_NAME}-static
    RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_PREFIX}/include
    )

install(FILES ${INCLUDE_FILES} DESTINATION
    ${CMAKE_INSTALL_PREFIX}/include/${PROJECT_NAME})
