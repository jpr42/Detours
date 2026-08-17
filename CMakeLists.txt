##############################################################################
##
##  CMakeLists.txt for Detours.
##
##  Microsoft Research Detours Package
##
##  Copyright (c) Microsoft Corporation.  All rights reserved.
##
##  This is provided alongside (not as a replacement for) the NMake build
##  driven by system.mak / src/Makefile / tests/Makefile.
##

cmake_minimum_required(VERSION 3.28.3)

project(Detours
    VERSION 4.0.1
    DESCRIPTION "Detours is a software package for monitoring and instrumenting API calls on Windows."
    HOMEPAGE_URL "https://github.com/microsoft/Detours"
    LANGUAGES CXX
)

enable_language(RC)
enable_language(ASM_MASM)

if(NOT WIN32)
    message(FATAL_ERROR "Detours only targets Windows; see https://github.com/microsoft/Detours/wiki/FAQ")
endif()

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
set(CMAKE_COMPILE_WARNING_AS_ERROR ON)

# CMake's MSVC Debug default injects /RTC1 (runtime checks), which has no equivalent in the
# original NMake build (its "Debug" is just the DETOUR_DEBUG=1 tracing macro, not a compiler
# runtime-checks mode) and actively conflicts with samples that need /Zl (no default CRT lib
# reference) or a non-default optimization level.
string(REPLACE "/RTC1" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")

option(DETOURS_BUILD_TESTS "Build the Detours unit test suite" ${PROJECT_IS_TOP_LEVEL})
option(DETOURS_BUILD_SAMPLES "Build the Detours samples" OFF)
option(DETOURS_INSTALL "Generate install/export rules for the Detours package" ${PROJECT_IS_TOP_LEVEL})

add_library(detours STATIC)
add_library(Microsoft::detours ALIAS detours)
add_subdirectory(src)

if(DETOURS_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

if(DETOURS_BUILD_SAMPLES)
    add_subdirectory(samples)
endif()

if(DETOURS_INSTALL)
    include(GNUInstallDirs)
    include(CMakePackageConfigHelpers)

    install(TARGETS detours
        EXPORT DetoursTargets
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        FILE_SET HEADERS DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )

    install(EXPORT DetoursTargets
        FILE DetoursTargets.cmake
        NAMESPACE Microsoft::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Detours
    )

    configure_package_config_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/DetoursConfig.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/DetoursConfig.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Detours
    )

    write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/DetoursConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/DetoursConfig.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/DetoursConfigVersion.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Detours
    )
endif()
