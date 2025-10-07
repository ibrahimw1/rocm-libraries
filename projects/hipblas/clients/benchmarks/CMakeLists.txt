# Copyright (C) 2016-2025 Advanced Micro Devices, Inc. All rights reserved.
# SPDX-License-Identifier: MIT

# Create benchmark executable
add_executable(hipblas-bench)

# Add benchmark sources
target_sources(
    hipblas-bench
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/client.cpp"
)

# Add benchmark-specific sources (clients_common.cpp is only used in benchmarks)
target_sources(
    hipblas-bench
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/../common/clients_common.cpp"
)

# Add common client sources via OBJECT library
target_sources(
    hipblas-bench
    PRIVATE $<TARGET_OBJECTS:hipblas-common-client>
)

# Add Fortran objects on non-Windows platforms
if(HIPBLAS_ENABLE_FORTRAN AND NOT WIN32)
    target_sources(
        hipblas-bench
        PRIVATE $<TARGET_OBJECTS:hipblas_fortran_client>
    )
endif()

# Target compile features
target_compile_features(hipblas-bench PRIVATE cxx_std_17)
target_compile_features(hipblas-bench PRIVATE cxx_static_assert cxx_nullptr cxx_auto_type)

# Include directories
target_include_directories(
    hipblas-bench
    PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
)

# External headers as SYSTEM
target_include_directories(
    hipblas-bench
    SYSTEM PRIVATE
        $<BUILD_INTERFACE:${HIP_INCLUDE_DIRS}>
        $<BUILD_INTERFACE:${CBLAS_INCLUDE_DIRS}>
        $<BUILD_INTERFACE:${BLAS_INCLUDE_DIR}>
        $<BUILD_INTERFACE:${BLIS_INCLUDE_DIR}>
        $<BUILD_INTERFACE:${FLAME_INCLUDE_DIR}>
)

# Compile definitions
target_compile_definitions(
    hipblas-bench 
    PRIVATE 
        HIPBLAS_BENCH
        ${COMMON_DEFINES}
        ${BLIS_DEFINES}
)

# Compile options
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    target_compile_options(hipblas-bench PRIVATE -mf16c)
endif()

# Link libraries
target_link_libraries(hipblas-bench PRIVATE roc::hipblas)
target_link_libraries(hipblas-bench PRIVATE GTest::gtest GTest::gtest_main)
target_link_libraries(hipblas-bench PRIVATE ${BLAS_LIBRARY})
target_link_libraries(hipblas-bench PRIVATE ${COMMON_LINK_LIBS})

if(HIPBLAS_ENABLE_FORTRAN AND NOT WIN32)
    target_link_libraries(hipblas-bench PRIVATE hipblas_fortran_client)
endif()

if(NOT WIN32)
    target_link_libraries(hipblas-bench PRIVATE stdc++fs)
endif()

# Platform-specific linking
if(HIPBLAS_ENABLE_HIP)
    target_link_libraries(hipblas-bench PRIVATE hip::host)
    
    if(CUSTOM_TARGET)
        target_link_libraries(hipblas-bench PRIVATE hip::${CUSTOM_TARGET})
    endif()
elseif(HIPBLAS_ENABLE_CUDA)
    target_include_directories(
        hipblas-bench
        SYSTEM PRIVATE
            $<BUILD_INTERFACE:${CUDA_INCLUDE_DIRS}>
    )
    target_link_libraries(hipblas-bench PRIVATE ${CUDA_LIBRARIES})
endif()

# Apply OpenMP settings
apply_omp_settings(hipblas-bench)

# Target properties
set_target_properties(
    hipblas-bench PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
    LINKER_LANGUAGE CXX
    RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging"
)

# Add dependency on common client files
add_dependencies(hipblas-bench hipblas-clients-common)

# Install benchmark executable
rocm_install(
    TARGETS hipblas-bench 
    COMPONENT benchmarks
    DESTINATION "${CMAKE_INSTALL_BINDIR}"
)
