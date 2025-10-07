# ########################################################################
# Copyright (C) 2016-2024 Advanced Micro Devices, Inc. All rights reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell cop-
# ies of the Software, and to permit persons to whom the Software is furnished
# to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IM-
# PLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
# FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
# COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
# IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNE-
# CTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
# ########################################################################

# Helper cmake script to automate building dependencies for hipblas
# This script can be invoked manually by the user with 'cmake -P'

cmake_minimum_required(VERSION 3.16)

# The superbuild does not build anything itself; all compiling is done in external projects
project(hipblas-dependencies NONE)

# Set default install prefix if not specified
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  if(WIN32)
    set(CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/package" CACHE PATH "Install path prefix" FORCE)
  else()
    set(CMAKE_INSTALL_PREFIX "/opt/rocm" CACHE PATH "Install path prefix" FORCE)
  endif()
endif()

# Set the default build type to Release
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build" FORCE)
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

# Options for controlling which dependencies to build
option(HIPBLAS_DEPS_BUILD_GTEST "Download and build googletest library" ON)
option(HIPBLAS_DEPS_BUILD_LAPACK "Download and build lapack library" ON)

# Add module paths
list(APPEND CMAKE_MODULE_PATH 
  "${CMAKE_CURRENT_SOURCE_DIR}" 
  "${CMAKE_CURRENT_SOURCE_DIR}/../cmake"
)

# This module scrapes the CMakeCache.txt file and attempts to get all the cli options the user specified to cmake invocation
include(get-cli-arguments)

# Include ExternalProject for dependency management
include(ExternalProject)

# List to track all dependencies
set(HIPBLAS_DEPENDENCIES)

# Build GTest if requested
if(HIPBLAS_DEPS_BUILD_GTEST)
  include(external-gtest)
  list(APPEND HIPBLAS_DEPENDENCIES googletest)
endif()

# Build LAPACK if requested
if(HIPBLAS_DEPS_BUILD_LAPACK)
  include(external-lapack)
  list(APPEND HIPBLAS_DEPENDENCIES lapack)
endif()

# Create a custom target that builds all dependencies
if(HIPBLAS_DEPENDENCIES)
  # Use add_custom_target to create an 'install' target at the superbuild level
  # This is safe because this is a NONE project (no language enabled)
  add_custom_target(install-dependencies
    COMMENT "Installing hipBLAS dependencies"
    DEPENDS ${HIPBLAS_DEPENDENCIES}
  )
  
  # For backward compatibility, also provide 'install' target
  # Note: This requires CMP0037 OLD policy
  if(POLICY CMP0037)
    cmake_policy(SET CMP0037 OLD)
  endif()
  
  add_custom_target(install
    COMMENT "Installing hipBLAS dependencies (legacy target)"
    DEPENDS install-dependencies
  )
else()
  message(STATUS "No dependencies selected to build")
endif()

# Print summary
message(STATUS "")
message(STATUS "hipBLAS Dependencies Configuration Summary:")
message(STATUS "  Install Prefix: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "  Build Type: ${CMAKE_BUILD_TYPE}")
message(STATUS "  Build GTest: ${HIPBLAS_DEPS_BUILD_GTEST}")
message(STATUS "  Build LAPACK: ${HIPBLAS_DEPS_BUILD_LAPACK}")
message(STATUS "")
