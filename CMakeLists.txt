cmake_minimum_required(VERSION 3.17.0 FATAL_ERROR)
project(pianobooster)

# allow using non-default configuration
set(CONFIGURATION_PACKAGE_SUFFIX
    ""
    CACHE STRING "sets the suffix for find_package() calls to packages configured via c++utilities")
set(CONFIGURATION_PACKAGE_SUFFIX_QTUTILITIES
    "${CONFIGURATION_PACKAGE_SUFFIX}"
    CACHE STRING "sets the suffix for qtutilities")
set(CONFIGURATION_PACKAGE_SUFFIX_QTFORKAWESOME
    "${CONFIGURATION_PACKAGE_SUFFIX_QTUTILITIES}"
    CACHE STRING "sets the suffix for qtforkawesome")
set(PACKAGE_NAMESPACE
    ""
    CACHE STRING "sets the prefix for find_package() calls to packages configured via c++utilities")
if (PACKAGE_NAMESPACE)
    set(PACKAGE_NAMESPACE_PREFIX "${PACKAGE_NAMESPACE}-")
endif ()

add_subdirectory(src build)
