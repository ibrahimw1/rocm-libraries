# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

target_sources(hipblas-bench
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/client.cpp"
)

target_compile_definitions(hipblas-bench PRIVATE
    HIPBLAS_BENCH
    HIPBLAS_BFLOAT16_CLASS
    ROCM_USE_FLOAT16
    HIPBLAS_NO_DEPRECATED_WARNINGS
)

target_compile_features(hipblas-bench PRIVATE cxx_std_17)

# F16C intrinsics flag - required for float->half conversion
target_compile_options(hipblas-bench PRIVATE -mf16c)

# Link against common client library
target_link_libraries(hipblas-bench
    PRIVATE
        hipblas-clients-common
        GTest::gtest
        GTest::gtest_main
        ${BLAS_LIBRARIES}
)

if(HIPBLAS_ENABLE_FORTRAN)
    target_sources(hipblas-bench PRIVATE $<TARGET_OBJECTS:hipblas_fortran_client>)
    target_link_libraries(hipblas-bench PRIVATE hipblas_fortran_client)
endif()

if(NOT WIN32)
    target_link_libraries(hipblas-bench PRIVATE stdc++fs)
endif()

if(HIP_PLATFORM STREQUAL "amd")
    target_link_libraries(hipblas-bench PRIVATE hip::host)
else()
    target_link_libraries(hipblas-bench PRIVATE ${CUDA_LIBRARIES})
endif()

set_target_properties(hipblas-bench PROPERTIES
    CXX_EXTENSIONS OFF
    RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging"
)

add_dependencies(hipblas-bench hipblas-clients-common-files)
