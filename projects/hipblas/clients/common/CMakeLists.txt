# Copyright (C) 2016-2025 Advanced Micro Devices, Inc. All rights reserved.
# SPDX-License-Identifier: MIT

# Create OBJECT library for common client code
add_library(hipblas-common-client OBJECT)

# Add common sources used by all clients
target_sources(
    hipblas-common-client
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/utility.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/cblas_interface.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/norm.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/unit.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/near.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/arg_check.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/argument_model.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_arguments.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_parse_data.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_datatype2string.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/host_alloc.cpp"
)

# Conditionally add BLIS interface if BLIS is being used
if(DEFINED BLIS_INCLUDE_DIR)
    target_sources(
        hipblas-common-client
        PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/blis_interface.cpp"
    )
endif()

# Target compile features
target_compile_features(hipblas-common-client PRIVATE cxx_std_17)

# Include directories
target_include_directories(
    hipblas-common-client
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
)

# External headers as SYSTEM
target_include_directories(
    hipblas-common-client
    SYSTEM PRIVATE
        $<BUILD_INTERFACE:${HIP_INCLUDE_DIRS}>
        $<BUILD_INTERFACE:${CBLAS_INCLUDE_DIRS}>
        $<BUILD_INTERFACE:${BLAS_INCLUDE_DIR}>
        $<BUILD_INTERFACE:${BLIS_INCLUDE_DIR}>
        $<BUILD_INTERFACE:${FLAME_INCLUDE_DIR}>
)

# Compile definitions
target_compile_definitions(
    hipblas-common-client 
    PRIVATE 
        ${COMMON_DEFINES}
        ${BLIS_DEFINES}
)

# Compile options
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    target_compile_options(hipblas-common-client PRIVATE -mf16c)
endif()
