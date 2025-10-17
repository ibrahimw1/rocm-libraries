# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

# Common utility source for all samples
set(samples_common_source "${CMAKE_CURRENT_SOURCE_DIR}/../common/utility.cpp")

# Sample: sscal
add_executable(hipblas-example-sscal)
target_sources(hipblas-example-sscal
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_sscal.cpp"
        ${samples_common_source}
)

# Sample: scal-ex
add_executable(hipblas-example-scal-ex)
target_sources(hipblas-example-scal-ex
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_scal_ex.cpp"
        ${samples_common_source}
)

# Sample: sgemm
add_executable(hipblas-example-sgemm)
target_sources(hipblas-example-sgemm
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_sgemm.cpp"
        ${samples_common_source}
)

# Sample: strmm
add_executable(hipblas-example-strmm)
target_sources(hipblas-example-strmm
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_strmm.cpp"
        ${samples_common_source}
)

# Sample: sgemm-strided-batched
add_executable(hipblas-example-sgemm-strided-batched)
target_sources(hipblas-example-sgemm-strided-batched
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_sgemm_strided_batched.cpp"
        ${samples_common_source}
)

# Sample: C example
add_executable(hipblas-example-c)
target_sources(hipblas-example-c
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_c.c"
        ${samples_common_source}
)
# Test C99 compatibility
set_source_files_properties(
    "${CMAKE_CURRENT_SOURCE_DIR}/example_c.c"
    PROPERTIES
        LANGUAGE CXX
        COMPILE_FLAGS "-xc -std=c99"
)
# hipcc needs to find hip includes for C example
target_compile_options(hipblas-example-c PRIVATE -I${HIP_INCLUDE_DIRS})

# Sample: hip-complex-her2
add_executable(hipblas-example-hip-complex-her2)
target_sources(hipblas-example-hip-complex-her2
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_hip_complex_her2.cpp"
        ${samples_common_source}
)

# Sample: hgemm-half
add_executable(hipblas-example-hgemm-half)
target_sources(hipblas-example-hgemm-half
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_hgemm_hip_half.cpp"
        ${samples_common_source}
)
target_compile_definitions(hipblas-example-hgemm-half PRIVATE HIPBLAS_USE_HIP_HALF)
if(WIN32)
    # HIP on Windows: xhip required with clang++ to get __half defined
    target_compile_options(hipblas-example-hgemm-half PRIVATE -xhip)
endif()

# Sample: gemmEx
add_executable(hipblas-example-gemmEx)
target_sources(hipblas-example-gemmEx
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/example_gemm_ex.cpp"
        ${samples_common_source}
)
target_compile_definitions(hipblas-example-gemmEx PRIVATE HIPBLAS_USE_HIP_HALF)
if(WIN32)
    target_compile_options(hipblas-example-gemmEx PRIVATE -xhip)
endif()

# Compiler-specific sample: hgemm (requires hipcc or amdclang++)
if(CMAKE_CXX_COMPILER MATCHES ".*/hipcc$" OR CMAKE_CXX_COMPILER MATCHES ".*/amdclang\\+\\+$")
    add_executable(hipblas-example-hgemm)
    target_sources(hipblas-example-hgemm
        PRIVATE
            "${CMAKE_CURRENT_SOURCE_DIR}/example_hgemm.cpp"
            ${samples_common_source}
    )
endif()

# Platform-specific sample: bfdot (AMD only)
if(HIP_PLATFORM STREQUAL "amd")
    add_executable(hipblas-example-bfdot-hip-bfloat16)
    target_sources(hipblas-example-bfdot-hip-bfloat16
        PRIVATE
            "${CMAKE_CURRENT_SOURCE_DIR}/example_bfdot_hip_bfloat16.cpp"
            ${samples_common_source}
    )
    target_compile_definitions(hipblas-example-bfdot-hip-bfloat16
        PRIVATE HIPBLAS_USE_HIP_BFLOAT16
    )
endif()

# Fortran samples (non-Windows only)
if(HIPBLAS_ENABLE_FORTRAN AND NOT WIN32)
    add_executable(hipblas-example-sscal-fortran)
    target_sources(hipblas-example-sscal-fortran
        PRIVATE
            "${CMAKE_CURRENT_SOURCE_DIR}/example_sscal_fortran.F90"
            $<TARGET_OBJECTS:hipblas_fortran_client>
    )
    
    add_executable(hipblas-example-gemmEx-fortran)
    target_sources(hipblas-example-gemmEx-fortran
        PRIVATE
            "${CMAKE_CURRENT_SOURCE_DIR}/example_gemm_ex_fortran.F90"
            $<TARGET_OBJECTS:hipblas_fortran_client>
    )
endif()

# Configure all sample targets with common properties
set(all_samples
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

if(CMAKE_CXX_COMPILER MATCHES ".*/hipcc$" OR CMAKE_CXX_COMPILER MATCHES ".*/amdclang\\+\\+$")
    list(APPEND all_samples hipblas-example-hgemm)
endif()

if(HIP_PLATFORM STREQUAL "amd")
    list(APPEND all_samples hipblas-example-bfdot-hip-bfloat16)
endif()

if(HIPBLAS_ENABLE_FORTRAN AND NOT WIN32)
    list(APPEND all_samples
        hipblas-example-sscal-fortran
        hipblas-example-gemmEx-fortran
    )
endif()

# Apply common configuration to all samples
foreach(sample_target IN LISTS all_samples)
    target_compile_features(${sample_target} PRIVATE cxx_std_17)
    
    # F16C and other common flags
    target_compile_options(${sample_target} PRIVATE -mf16c)
    target_compile_definitions(${sample_target} PRIVATE
        HIPBLAS_BFLOAT16_CLASS
        HIPBLAS_NO_DEPRECATED_WARNINGS
    )
    
    target_include_directories(${sample_target}
        PRIVATE
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
    )
    
    target_link_libraries(${sample_target} PRIVATE roc::hipblas)
    
    if(HIP_PLATFORM STREQUAL "amd")
        target_link_libraries(${sample_target} PRIVATE hip::host)
    else()
        target_compile_definitions(${sample_target} PRIVATE
            __HIP_PLATFORM_NVCC__
            __HIP_PLATFORM_NVIDIA__
        )
        target_link_libraries(${sample_target} PRIVATE ${CUDA_LIBRARIES})
    endif()
    
    set_target_properties(${sample_target} PROPERTIES
        LINKER_LANGUAGE CXX
        CXX_STANDARD 17
        CXX_STANDARD_REQUIRED ON
        CXX_EXTENSIONS ON
        RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging"
    )
    
    # Install each sample
    rocm_install(TARGETS ${sample_target} COMPONENT samples)
endforeach()
