# Copyright (C) 2016-2025 Advanced Micro Devices, Inc. All rights reserved.
# SPDX-License-Identifier: MIT

# Print verbose build information if requested
if(HIPBLAS_ENABLE_VERBOSE)
    message(STATUS "hipBLAS version: ${PROJECT_VERSION}")
    message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
    message(STATUS "Shared libs: ${HIPBLAS_BUILD_SHARED_LIBS}")
    message(STATUS "Install prefix: ${CMAKE_INSTALL_PREFIX}")
endif()

# Get the git hash of the hipBLAS branch
find_package(Git REQUIRED)

execute_process(
    COMMAND "${GIT_EXECUTABLE}" rev-parse HEAD
    WORKING_DIRECTORY "${CMAKE_CURRENT_LIST_DIR}"
    OUTPUT_VARIABLE GIT_HASH_HIPBLAS
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

set(hipblas_VERSION_COMMIT_ID "${GIT_HASH_HIPBLAS}")
message(STATUS "Building hipBLAS commit: ${hipblas_VERSION_COMMIT_ID}")

# Configure version header
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas-version.h.in" 
    "${PROJECT_BINARY_DIR}/include/hipblas/hipblas-version.h"
)

# Copy public headers to build directory
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas.h" 
    "${PROJECT_BINARY_DIR}/include/hipblas/hipblas.h" 
    COPYONLY
)

# Set Fortran module output directory
if(NOT WIN32 AND HIPBLAS_ENABLE_FORTRAN)
    set(CMAKE_Fortran_MODULE_DIRECTORY "${PROJECT_BINARY_DIR}/include")
endif()

# Public header files
set(hipblas_headers_public
    include/hipblas.h
    ${PROJECT_BINARY_DIR}/include/hipblas/hipblas-version.h
)

# Configure target include directories
target_include_directories(
    hipblas
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include/hipblas>
        $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Generate export header for symbol visibility
include(GenerateExportHeader)
generate_export_header(
    hipblas 
    EXPORT_FILE_NAME "${PROJECT_BINARY_DIR}/include/hipblas/hipblas-export.h"
)

# Add subdirectories to populate target
add_subdirectory(include)
add_subdirectory(src)

# Source file grouping for IDEs
source_group("Header Files\\Public" FILES ${hipblas_headers_public})

# Set runtime output directory for Windows DLL
if(WIN32)
    set_target_properties(
        hipblas PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging"
    )
endif()
