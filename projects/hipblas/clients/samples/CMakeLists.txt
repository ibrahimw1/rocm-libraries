# Copyright (C) 2016-2025 Advanced Micro Devices, Inc. All rights reserved.
# SPDX-License-Identifier: MIT

# Common utility source for all samples
set(hipblas_samples_common "${CMAKE_CURRENT_SOURCE_DIR}/../common/utility.cpp")

# Define all sample executables
add_executable(hipblas-example-sscal)
add_executable(hipblas-example-scal-ex)
add_executable(hipblas-example-sgemm)
add_executable(hipblas-example-strmm)
add_executable(hipblas-example-sgemm-strided-batched)
add_executable(hipblas-example-c)
add_executable(hipblas-example-hip-complex-her2)
add_executable(hipblas-example-hgemm-half)
add_executable(hipblas-example-gemmEx)

# List of all sample targets
set(hipblas_sample_targets
    hipblas-example-sscal
    hipblas-example-scal-ex
    hipblas-example-sgemm
    hipblas-example-strmm
    hipblas-example-sgemm-strided-batched
    hipblas-example-c
    hipblas-example-hip-complex-her2
    hipblas-example-hgemm-half
    hipblas-example-gemmEx
)

# Add sources to each sample
target_sources(hipblas-example-sscal PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_sscal.cpp" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-scal-ex PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_scal_ex.cpp" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-sgemm PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_sgemm.cpp" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-strmm PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_strmm.cpp" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-sgemm-strided-batched PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_sgemm_strided_batched.cpp" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-c PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_c.c" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-hip-complex-her2 PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_hip_complex_her2.cpp" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-hgemm-half PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_hgemm_hip_half.cpp" 
    ${hipblas_samples_common}
)

target_sources(hipblas-example-gemmEx PRIVATE 
    "${CMAKE_CURRENT_SOURCE_DIR}/example_gemm_ex.cpp" 
    ${hipblas_samples_common}
)

# Add hipcc/amdclang++-specific sample
if(CMAKE_CXX_COMPILER MATCHES ".*/hipcc$" OR CMAKE_CXX_COMPILER MATCHES ".*/amdclang\\+\\+$")
    add_executable(hipblas-example-hgemm)
    target_sources(hipblas-example-hgemm PRIVATE 
        "${CMAKE_CURRENT_SOURCE_DIR}/example_hgemm.cpp" 
        ${hipblas_samples_common}
    )
    list(APPEND hipblas_sample_targets hipblas-example-hgemm)
endif()

# Add Fortran samples on non-Windows platforms
if(HIPBLAS_ENABLE_FORTRAN AND NOT WIN32)
    add_executable(hipblas-example-sscal-fortran)
    target_sources(hipblas-example-sscal-fortran PRIVATE 
        "${CMAKE_CURRENT_SOURCE_DIR}/example_sscal_fortran.F90"
        $<TARGET_OBJECTS:hipblas_fortran_client>
    )
    
    add_executable(hipblas-example-gemmEx-fortran)
    target_sources(hipblas-example-gemmEx-fortran PRIVATE 
        "${CMAKE_CURRENT_SOURCE_DIR}/example_gemm_ex_fortran.F90"
        $<TARGET_OBJECTS:hipblas_fortran_client>
    )
    
    list(APPEND hipblas_sample_targets 
        hipblas-example-sscal-fortran 
        hipblas-example-gemmEx-fortran
    )
endif()

# Add HIP-specific sample
if(HIPBLAS_ENABLE_HIP)
    add_executable(hipblas-example-bfdot-hip-bfloat16)
    target_sources(hipblas-example-bfdot-hip-bfloat16 PRIVATE 
        "${CMAKE_CURRENT_SOURCE_DIR}/example_bfdot_hip_bfloat16.cpp" 
        ${hipblas_samples_common}
    )
    target_compile_definitions(hipblas-example-bfdot-hip-bfloat16 PRIVATE HIPBLAS_USE_HIP_BFLOAT16)
    list(APPEND hipblas_sample_targets hipblas-example-bfdot-hip-bfloat16)
endif()

# Configure C example for C99 compatibility
set_source_files_properties(
    "${CMAKE_CURRENT_SOURCE_DIR}/example_c.c" 
    PROPERTIES 
        LANGUAGE CXX
        COMPILE_FLAGS "-xc -std=c99"
)

# Configure examples that use half precision
target_compile_definitions(hipblas-example-hgemm-half PRIVATE HIPBLAS_USE_HIP_HALF)
target_compile_definitions(hipblas-example-gemmEx PRIVATE HIPBLAS_USE_HIP_HALF)

# Windows requires -xhip flag for __half support
if(WIN32)
    target_compile_options(hipblas-example-hgemm-half PRIVATE -xhip)
    target_compile_options(hipblas-example-gemmEx PRIVATE -xhip)
endif()

# Find hipblas if not already a target
if(NOT TARGET hipblas)
    find_package(hipblas REQUIRED CONFIG)
endif()

# Common configuration for all samples
foreach(exe IN LISTS hipblas_sample_targets)
    # Target compile features
    target_compile_features(${exe} PRIVATE cxx_std_17)
    
    # Include directories
    target_include_directories(
        ${exe}
        PRIVATE
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
    )
    
    # External headers as SYSTEM
    target_include_directories(
        ${exe}
        SYSTEM PRIVATE
            $<BUILD_INTERFACE:${HIP_INCLUDE_DIRS}>
    )
    
    # Compile options and definitions
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        target_compile_options(${exe} PRIVATE -mf16c)
    endif()
    
    target_compile_definitions(${exe} PRIVATE 
        HIPBLAS_BFLOAT16_CLASS
        HIPBLAS_NO_DEPRECATED_WARNINGS
    )
    
    # Link libraries
    target_link_libraries(${exe} PRIVATE roc::hipblas)
    
    # Platform-specific configuration
    if(HIPBLAS_ENABLE_HIP)
        target_link_libraries(${exe} PRIVATE hip::host)
        
        if(CUSTOM_TARGET)
            target_link_libraries(${exe} PRIVATE hip::${CUSTOM_TARGET})
        endif()
    elseif(HIPBLAS_ENABLE_CUDA)
        target_compile_definitions(${exe} PRIVATE __HIP_PLATFORM_NVCC__ __HIP_PLATFORM_NVIDIA__)
        
        target_include_directories(
            ${exe}
            SYSTEM PRIVATE
                $<BUILD_INTERFACE:${CUDA_INCLUDE_DIRS}>
        )
        
        target_link_libraries(${exe} PRIVATE ${CUDA_LIBRARIES})
    endif()
    
    # Target properties
    set_target_properties(
        ${exe} PROPERTIES
        LINKER_LANGUAGE CXX
        CXX_STANDARD 17
        CXX_STANDARD_REQUIRED ON
        CXX_EXTENSIONS ON
        RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging"
    )
    
    # Install sample
    rocm_install(
        TARGETS ${exe} 
        COMPONENT samples
        DESTINATION "${CMAKE_INSTALL_BINDIR}"
    )
endforeach()

# Special handling for C example - needs HIP include for hipcc
target_compile_options(hipblas-example-c PRIVATE -I${HIP_INCLUDE_DIRS})
