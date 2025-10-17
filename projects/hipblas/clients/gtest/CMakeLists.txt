# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

target_sources(hipblas-test
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_gtest_main.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_test.cpp"
        # Auxiliary tests
        "${CMAKE_CURRENT_SOURCE_DIR}/auxil/auxiliary_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/auxil/set_get_mode_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/auxil/set_get_matrix_vector_gtest.cpp"
        # BLAS1 tests
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/asum_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/axpy_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/copy_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/dot_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/iamaxmin_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/nrm2_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/rot_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/scal_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas1/swap_gtest.cpp"
        # BLAS2 tests
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/gbmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/gemv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/ger_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/hbmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/hemv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/her_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/her2_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/hpmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/hpr_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/hpr2_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/sbmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/spmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/spr_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/spr2_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/symv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/syr_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/syr2_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/tbmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/tbsv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/tpmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/tpsv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/trmv_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas2/trsv_gtest.cpp"
        # BLAS3 tests
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/dgmm_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/gemm_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/hemm_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/geam_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/herk_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/her2k_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/herkx_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/symm_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/syrk_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/syr2k_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/syrkx_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/trsm_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/trmm_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas3/trtri_gtest.cpp"
        # BLAS_EX tests
        "${CMAKE_CURRENT_SOURCE_DIR}/blas_ex/axpy_ex_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas_ex/dot_ex_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas_ex/nrm2_ex_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas_ex/rot_ex_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas_ex/scal_ex_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas_ex/trsm_ex_gtest.cpp"
        "${CMAKE_CURRENT_SOURCE_DIR}/blas_ex/gemm_ex_gtest.cpp"
)

# Solver tests if enabled
if(HIPBLAS_ENABLE_SOLVER)
    target_sources(hipblas-test
        PRIVATE
            "${CMAKE_CURRENT_SOURCE_DIR}/solver/getrf_gtest.cpp"
            "${CMAKE_CURRENT_SOURCE_DIR}/solver/getrs_gtest.cpp"
            "${CMAKE_CURRENT_SOURCE_DIR}/solver/getri_gtest.cpp"
            "${CMAKE_CURRENT_SOURCE_DIR}/solver/geqrf_gtest.cpp"
            "${CMAKE_CURRENT_SOURCE_DIR}/solver/gels_gtest.cpp"
    )
endif()

# Fortran object files if enabled
if(HIPBLAS_ENABLE_FORTRAN)
    target_sources(hipblas-test PRIVATE $<TARGET_OBJECTS:hipblas_fortran_client>)
endif()

target_compile_definitions(hipblas-test PRIVATE GOOGLE_TEST)
target_compile_features(hipblas-test PRIVATE cxx_std_17)

# F16C intrinsics flag - required for float->half conversion
target_compile_options(hipblas-test PRIVATE -mf16c)

# Link against common client library
target_link_libraries(hipblas-test
    PRIVATE
        hipblas-clients-common
        GTest::gtest
        GTest::gtest_main
        ${BLAS_LIBRARIES}
)

if(HIPBLAS_ENABLE_FORTRAN)
    target_link_libraries(hipblas-test PRIVATE hipblas_fortran_client)
endif()

if(NOT WIN32)
    target_link_libraries(hipblas-test PRIVATE stdc++fs)
endif()

if(HIP_PLATFORM STREQUAL "amd")
    target_link_libraries(hipblas-test PRIVATE hip::host)
else()
    target_link_libraries(hipblas-test PRIVATE ${CUDA_LIBRARIES})
endif()

# Windows-specific DLL copying
if(WIN32)
    file(GLOB third_party_dlls
        LIST_DIRECTORIES OFF
        CONFIGURE_DEPENDS
        ${LAPACK_DIR}/bin/*.dll
        ${BLIS_DIR}/lib/*.dll
        ${OPENBLAS_DIR}/bin/*.dll
        ${HIP_DIR}/bin/amd*.dll
        ${HIP_DIR}/bin/hiprt*.dll
        ${HIP_DIR}/bin/hipinfo.exe
        ${ROCBLAS_PATH}/bin/rocblas*.dll
        ${ROCSOLVER_PATH}/bin/rocsolver*.dll
        ${CMAKE_SOURCE_DIR}/rtest.*
        C:/Windows/System32/libomp140*.dll
    )
    foreach(file_i ${third_party_dlls})
        add_custom_command(
            TARGET hipblas-test POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy ${file_i} ${PROJECT_BINARY_DIR}/staging/
        )
    endforeach()
    
    if(DEFINED ROCBLAS_PATH)
        add_custom_command(
            TARGET hipblas-test POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_directory
                ${ROCBLAS_PATH}/bin/rocblas/library/
                ${PROJECT_BINARY_DIR}/staging/library/
        )
    else()
        message(WARNING "ROCBLAS_PATH not set. rocBLAS Tensile runtime kernels will not be copied.")
    endif()
endif()

set_target_properties(hipblas-test PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
    CXX_EXTENSIONS OFF
    RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/staging"
)

# Generate test data file
set(HIPBLAS_TEST_DATA "${PROJECT_BINARY_DIR}/staging/hipblas_gtest.data")

set(HIPBLAS_AUX_YAML_DATA
    auxil/set_get_matrix_vector_gtest.yaml
    auxil/set_get_mode_gtest.yaml
)

set(HIPBLAS_L1_YAML_DATA
    blas1/asum_gtest.yaml
    blas1/axpy_gtest.yaml
    blas1/copy_gtest.yaml
    blas1/dot_gtest.yaml
    blas1/iamaxmin_gtest.yaml
    blas1/nrm2_gtest.yaml
    blas1/rot_gtest.yaml
    blas1/scal_gtest.yaml
    blas1/swap_gtest.yaml
)

set(HIPBLAS_L2_YAML_DATA
    blas2/gbmv_gtest.yaml
    blas2/gemv_gtest.yaml
    blas2/ger_gtest.yaml
    blas2/hbmv_gtest.yaml
    blas2/hemv_gtest.yaml
    blas2/her_gtest.yaml
    blas2/her2_gtest.yaml
    blas2/hpmv_gtest.yaml
    blas2/hpr_gtest.yaml
    blas2/hpr2_gtest.yaml
    blas2/sbmv_gtest.yaml
    blas2/spmv_gtest.yaml
    blas2/spr_gtest.yaml
    blas2/spr2_gtest.yaml
    blas2/symv_gtest.yaml
    blas2/syr_gtest.yaml
    blas2/syr2_gtest.yaml
    blas2/tbmv_gtest.yaml
    blas2/tbsv_gtest.yaml
    blas2/tpmv_gtest.yaml
    blas2/tpsv_gtest.yaml
    blas2/trmv_gtest.yaml
    blas2/trsv_gtest.yaml
)

set(HIPBLAS_L3_YAML_DATA
    blas3/dgmm_gtest.yaml
    blas3/geam_gtest.yaml
    blas3/gemm_gtest.yaml
    blas3/hemm_gtest.yaml
    blas3/herk_gtest.yaml
    blas3/her2k_gtest.yaml
    blas3/herkx_gtest.yaml
    blas3/symm_gtest.yaml
    blas3/syrk_gtest.yaml
    blas3/syr2k_gtest.yaml
    blas3/syrkx_gtest.yaml
    blas3/trmm_gtest.yaml
    blas3/trsm_gtest.yaml
    blas3/trtri_gtest.yaml
)

set(HIPBLAS_EX_YAML_DATA
    blas_ex/axpy_ex_gtest.yaml
    blas_ex/dot_ex_gtest.yaml
    blas_ex/nrm2_ex_gtest.yaml
    blas_ex/rot_ex_gtest.yaml
    blas_ex/scal_ex_gtest.yaml
    blas_ex/gemm_ex_gtest.yaml
    blas_ex/trsm_ex_gtest.yaml
)

if(HIPBLAS_ENABLE_SOLVER)
    set(HIPBLAS_SOLVER_YAML_DATA
        solver/gels_gtest.yaml
        solver/geqrf_gtest.yaml
        solver/getrf_gtest.yaml
        solver/getri_gtest.yaml
        solver/getrs_gtest.yaml
    )
endif()

add_custom_command(
    OUTPUT "${HIPBLAS_TEST_DATA}"
    COMMAND ${Python3_EXECUTABLE}
        "${CMAKE_CURRENT_SOURCE_DIR}/../common/hipblas_gentest.py"
        -I "${CMAKE_CURRENT_SOURCE_DIR}/../include"
        hipblas_gtest.yaml
        -o "${HIPBLAS_TEST_DATA}"
    DEPENDS
        "${CMAKE_CURRENT_SOURCE_DIR}/../common/hipblas_gentest.py"
        "${CMAKE_CURRENT_SOURCE_DIR}/../include/hipblas_common.yaml"
        "${HIPBLAS_AUX_YAML_DATA}"
        "${HIPBLAS_L1_YAML_DATA}"
        "${HIPBLAS_L2_YAML_DATA}"
        "${HIPBLAS_L3_YAML_DATA}"
        "${HIPBLAS_EX_YAML_DATA}"
        "${HIPBLAS_SOLVER_YAML_DATA}"
        hipblas_gtest.yaml
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
)

add_custom_target(hipblas-test-data
    DEPENDS "${HIPBLAS_TEST_DATA}"
)

add_dependencies(hipblas-test hipblas-test-data hipblas-clients-common-files)
