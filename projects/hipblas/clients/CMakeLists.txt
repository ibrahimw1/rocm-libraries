# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

# Find required dependencies
find_package(Threads REQUIRED)
find_package(Python3 REQUIRED COMPONENTS Interpreter)

if(HIPBLAS_BUILD_TESTING OR HIPBLAS_ENABLE_BENCHMARKS)
    find_package(GTest REQUIRED)
    find_package(OpenMP REQUIRED)
    find_package(BLAS REQUIRED)
    
    if(HIPBLAS_ENABLE_BLIS)
        # Try to find BLIS, but don't fail if not found
        find_package(BLIS QUIET)
    endif()
    
    if(HIPBLAS_ENABLE_FORTRAN)
        find_package(LAPACK REQUIRED)
    endif()
endif()

# Configure Fortran modules if enabled
if(HIPBLAS_ENABLE_FORTRAN)
    set(CMAKE_Fortran_MODULE_DIRECTORY "${PROJECT_BINARY_DIR}/include/hipblas")
    
    add_library(hipblas_fortran OBJECT
        "${CMAKE_CURRENT_SOURCE_DIR}/../library/src/hipblas_module.f90"
    )
    
    if(HIPBLAS_ENABLE_SOLVER)
        add_library(hipblas_fortran_client STATIC
            "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_fortran_module.f90"
            $<TARGET_OBJECTS:hipblas_fortran>
        )
    else()
        add_library(hipblas_fortran_client STATIC
            "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_fortran_no_solver_module.f90"
            $<TARGET_OBJECTS:hipblas_fortran>
        )
    endif()
endif()

# Add common client library
add_subdirectory(common)

# Add client subdirectories
if(HIPBLAS_ENABLE_SAMPLES)
    add_subdirectory(samples)
endif()

if(HIPBLAS_BUILD_TESTING)
    add_subdirectory(gtest)
endif()

if(HIPBLAS_ENABLE_BENCHMARKS)
    add_subdirectory(benchmarks)
endif()

# Custom commands for copying YAML and Python files to staging
set(HIPBLAS_COMMON "${PROJECT_BINARY_DIR}/staging/hipblas_common.yaml")
add_custom_command(
    OUTPUT "${HIPBLAS_COMMON}"
    COMMAND ${CMAKE_COMMAND} -E copy
        "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_common.yaml"
        "${HIPBLAS_COMMON}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_common.yaml"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
)

set(HIPBLAS_TEMPLATE "${PROJECT_BINARY_DIR}/staging/hipblas_template.yaml")
add_custom_command(
    OUTPUT "${HIPBLAS_TEMPLATE}"
    COMMAND ${CMAKE_COMMAND} -E copy
        "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_template.yaml"
        "${HIPBLAS_TEMPLATE}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_template.yaml"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
)

set(HIPBLAS_SMOKE "${PROJECT_BINARY_DIR}/staging/hipblas_smoke.yaml")
add_custom_command(
    OUTPUT "${HIPBLAS_SMOKE}"
    COMMAND ${CMAKE_COMMAND} -E copy
        "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_smoke.yaml"
        "${HIPBLAS_SMOKE}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_smoke.yaml"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
)

set(HIPBLAS_GENTEST "${PROJECT_BINARY_DIR}/staging/hipblas_gentest.py")
add_custom_command(
    OUTPUT "${HIPBLAS_GENTEST}"
    COMMAND ${CMAKE_COMMAND} -E copy
        "${CMAKE_CURRENT_SOURCE_DIR}/common/hipblas_gentest.py"
        "${HIPBLAS_GENTEST}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/common/hipblas_gentest.py"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
)

set(HIPBLAS_CLIENTS_README "${PROJECT_BINARY_DIR}/staging/hipblas_clients_readme.txt")
add_custom_command(
    OUTPUT "${HIPBLAS_CLIENTS_README}"
    COMMAND ${CMAKE_COMMAND} -E copy
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_clients_readme.txt"
        "${HIPBLAS_CLIENTS_README}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_clients_readme.txt"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
)

add_custom_target(hipblas-clients-common-files
    DEPENDS
        "${HIPBLAS_COMMON}"
        "${HIPBLAS_TEMPLATE}"
        "${HIPBLAS_SMOKE}"
        "${HIPBLAS_GENTEST}"
        "${HIPBLAS_CLIENTS_README}"
)
