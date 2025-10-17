# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

target_sources(hipblas-clients-common
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/utility.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/cblas_interface.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/clients_common.cpp"
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

if(HIPBLAS_ENABLE_BLIS AND TARGET BLIS::BLIS)
    target_sources(hipblas-clients-common
        PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/blis_interface.cpp"
    )
    target_link_libraries(hipblas-clients-common PUBLIC BLIS::BLIS)
endif()

target_include_directories(hipblas-clients-common
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
)

target_compile_features(hipblas-clients-common PUBLIC cxx_std_17)

# F16C intrinsics flag - required for float->half conversion in type_utils.h
target_compile_options(hipblas-clients-common PUBLIC -mf16c)

# Client compile definitions
target_compile_definitions(hipblas-clients-common PUBLIC
    HIPBLAS_BFLOAT16_CLASS
    ROCM_USE_FLOAT16
    HIPBLAS_NO_DEPRECATED_WARNINGS
)

target_link_libraries(hipblas-clients-common
    PUBLIC
        roc::hipblas
        Threads::Threads
)

if(HIPBLAS_ENABLE_FORTRAN)
    target_link_libraries(hipblas-clients-common PUBLIC ${LAPACK_LIBRARIES})
endif()

if(TARGET OpenMP::OpenMP_CXX)
    target_link_libraries(hipblas-clients-common PUBLIC OpenMP::OpenMP_CXX)
endif()

if(HIP_PLATFORM STREQUAL "amd")
    target_link_libraries(hipblas-clients-common PUBLIC hip::host)
else()
    target_compile_definitions(hipblas-clients-common PUBLIC
        __HIP_PLATFORM_NVCC__
        __HIP_PLATFORM_NVIDIA__
    )
endif()
