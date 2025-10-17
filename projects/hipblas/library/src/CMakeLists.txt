# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

# Platform-specific source selection
if(HIP_PLATFORM STREQUAL "amd")
    set(platform_source "${CMAKE_CURRENT_SOURCE_DIR}/amd_detail/hipblas.cpp")
    # Verbose compilation flag - legacy requirement for QC purposes
    set_source_files_properties(
        "${CMAKE_CURRENT_SOURCE_DIR}/amd_detail/hipblas.cpp"
        PROPERTIES COMPILE_FLAGS "-v"
    )
else()
    set(platform_source "${CMAKE_CURRENT_SOURCE_DIR}/nvidia_detail/hipblas.cpp")
endif()

target_sources(hipblas
    PRIVATE
        ${platform_source}
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_auxiliary.cpp"
)

target_include_directories(hipblas
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/include"
)

# Apply coverage flags if enabled
if(HIPBLAS_ENABLE_COVERAGE)
    target_compile_options(hipblas PRIVATE
        -fprofile-instr-generate
        -fcoverage-mapping
    )
    target_link_options(hipblas PRIVATE -fprofile-instr-generate)
endif()
