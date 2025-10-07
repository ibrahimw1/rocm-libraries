# Copyright (C) 2016-2025 Advanced Micro Devices, Inc. All rights reserved.
# SPDX-License-Identifier: MIT

# Helper functions for finding BLAS/LAPACK
function(get_lapack lapack_lib lapack_inc)
    if(EXISTS "${BUILD_DIR}/deps/deps-install/lib/liblapack.a")
        set(lib "${BUILD_DIR}/deps/deps-install/lib/liblapack.a")
        set(inc "${BUILD_DIR}/deps/deps-install/include")
        set(${lapack_inc} ${inc} PARENT_SCOPE)
    else()
        find_package(LAPACK REQUIRED)
        set(lib "${LAPACK_LIBRARIES}")
    endif()
    set(${lapack_lib} ${lib} PARENT_SCOPE)
endfunction()

function(get_cblas cblas_libs cblas_inc)
    if(LINK_BLIS)
        if(EXISTS "${BUILD_DIR}/deps/blis/lib/libblis.a")
            set(libs ${BUILD_DIR}/deps/blis/lib/libblis.a)
            set(inc ${BUILD_DIR}/deps/blis/include/blis)
            set(${cblas_inc} ${inc} PARENT_SCOPE)
        elseif(EXISTS "/usr/local/lib/libblis.a")
            set(libs /usr/local/lib/libblis.a)
            set(inc /usr/local/include/blis)
            set(${cblas_inc} ${inc} PARENT_SCOPE)
        endif()
    else()
        if(EXISTS "${BUILD_DIR}/deps/deps-install/lib/libcblas.a" AND EXISTS "${BUILD_DIR}/deps/deps-install/lib/libblas.a")
            set(libs ${BUILD_DIR}/deps/deps-install/lib/libcblas.a ${BUILD_DIR}/deps/deps-install/lib/libblas.a)
            set(inc "${BUILD_DIR}/deps/deps-install/include")
            set(${cblas_inc} ${inc} PARENT_SCOPE)
        else()
            find_package(cblas REQUIRED CONFIG)
            set(libs ${CBLAS_LIBRARIES})
        endif()
    endif()
    set(${cblas_libs} ${libs} PARENT_SCOPE)
endfunction()

function(apply_omp_settings lib_target_)
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND TARGET OpenMP::OpenMP_CXX)
        set_target_properties(${lib_target_} PROPERTIES
            BUILD_RPATH "${HIP_CLANG_ROOT}/lib"
            INSTALL_RPATH "$ORIGIN/../llvm/lib"
        )
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND TARGET OpenMP::omp)
        set_target_properties(${lib_target_} PROPERTIES
            BUILD_RPATH "${HIP_CLANG_ROOT}/${openmp_LIB_DIR}"
            INSTALL_RPATH "$ORIGIN/../llvm/${openmp_LIB_DIR}"
        )
    endif()
endfunction()

# Build common client code library
if(HIPBLAS_BUILD_TESTING OR HIPBLAS_ENABLE_BENCHMARKS OR HIPBLAS_ENABLE_SAMPLES)
    add_subdirectory(common)
endif()

# Fortran client library setup
if(HIPBLAS_ENABLE_FORTRAN AND NOT WIN32)
    set(CMAKE_Fortran_MODULE_DIRECTORY "${PROJECT_BINARY_DIR}/include/hipblas")
    
    # Base Fortran module from library
    add_library(hipblas_fortran OBJECT)
    target_sources(
        hipblas_fortran
        PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/../library/src/hipblas_module.f90"
    )
    
    # Client-specific Fortran module
    if(HIPBLAS_ENABLE_SOLVER)
        add_library(hipblas_fortran_client STATIC)
        target_sources(
            hipblas_fortran_client
            PRIVATE 
                "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_fortran_module.f90"
                $<TARGET_OBJECTS:hipblas_fortran>
        )
    else()
        add_library(hipblas_fortran_client STATIC)
        target_sources(
            hipblas_fortran_client
            PRIVATE 
                "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_fortran_no_solver_module.f90"
                $<TARGET_OBJECTS:hipblas_fortran>
        )
    endif()
endif()

# Build samples if enabled
if(HIPBLAS_ENABLE_SAMPLES)
    add_subdirectory(samples)
endif()

# Setup for test and benchmark clients
if(HIPBLAS_BUILD_TESTING OR HIPBLAS_ENABLE_BENCHMARKS)
    # Find threading library
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)
    
    # Find OpenMP
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        find_package(OpenMP CONFIG PATHS "${HIP_CLANG_ROOT}/lib/cmake")
    endif()
    
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND TARGET OpenMP::omp)
        set(COMMON_LINK_LIBS "OpenMP::omp")
        message(STATUS "Found openmp-config.cmake at ${OpenMP_DIR}")
    else()
        find_package(OpenMP)
        if(TARGET OpenMP::OpenMP_CXX)
            set(COMMON_LINK_LIBS "OpenMP::OpenMP_CXX")
        endif()
    endif()
    
    if(TARGET Threads::Threads)
        list(APPEND COMMON_LINK_LIBS "Threads::Threads")
    endif()
    
    # Common compile definitions for clients
    set(COMMON_DEFINES 
        HIPBLAS_BFLOAT16_CLASS 
        ROCM_USE_FLOAT16 
        HIPBLAS_NO_DEPRECATED_WARNINGS
    )
    
    if(HIPBLAS_ENABLE_CUDA)
        list(APPEND COMMON_DEFINES __HIP_PLATFORM_NVCC__ __HIP_PLATFORM_NVIDIA__)
    endif()
    
    message(STATUS "CLIENT COMMON_DEFINES: ${COMMON_DEFINES}")
    message(STATUS "CLIENT COMMON LINK: ${COMMON_LINK_LIBS}")
    
    # Find BLAS/LAPACK
    if(NOT WIN32)
        if(LINK_BLIS)
            # Try to find AOCL BLIS libraries
            if(EXISTS "${BUILD_DIR}/deps/amd-blis/lib/ILP64/libflame.a" AND EXISTS "${BUILD_DIR}/deps/amd-blis/lib/ILP64/libblis-mt.a")
                set(BLAS_LIBRARY ${BUILD_DIR}/deps/amd-blis/lib/ILP64/libflame.a ${BUILD_DIR}/deps/amd-blis/lib/ILP64/libblis-mt.a)
                set(BLIS_INCLUDE_DIR ${BUILD_DIR}/deps/amd-blis/include/ILP64)
                list(APPEND COMMON_DEFINES "FLA_ENABLE_ILP64")
            elseif(EXISTS "/opt/AMD/aocl/aocl-linux-gcc-4.2.0/gcc/lib_ILP64/libflame.a")
                set(BLAS_LIBRARY 
                    -Wl,--allow-multiple-definition 
                    /opt/AMD/aocl/aocl-linux-gcc-4.2.0/gcc/lib_ILP64/libflame.a 
                    /opt/AMD/aocl/aocl-linux-gcc-4.2.0/gcc/lib_ILP64/libblis-mt.a 
                    /opt/AMD/aocl/aocl-linux-gcc-4.2.0/gcc/lib_ILP64/libaoclutils.a
                )
                set(BLIS_INCLUDE_DIR /opt/AMD/aocl/aocl-linux-gcc-4.2.0/gcc/include_ILP64/)
                list(APPEND COMMON_DEFINES "FLA_ENABLE_ILP64")
            else()
                get_lapack(LAPACK_LIB LAPACK_INCLUDE_DIR)
                get_cblas(BLAS_LIBRARY BLAS_INCLUDE_DIR)
                list(PREPEND BLAS_LIBRARY ${LAPACK_LIB})
            endif()
        else()
            get_lapack(LAPACK_LIB LAPACK_INCLUDE_DIR)
            get_cblas(BLAS_LIBRARY BLAS_INCLUDE_DIR)
            list(PREPEND BLAS_LIBRARY ${LAPACK_LIB})
        endif()
    else() # WIN32
        if(LINK_BLIS AND EXISTS "C:/Program Files/AMD/AOCL-Windows/amd-blis/lib/ILP64/AOCL-LibBlis-Win-MT.lib")
            set(BLAS_LIBRARY 
                "-l\"C:/Program Files/AMD/AOCL-Windows/amd-blis/lib/ILP64/AOCL-LibBlis-Win-MT\"" 
                "-l\"C:/Program Files/AMD/AOCL-Windows/amd-libflame/lib/ILP64/AOCL-LibFlame-Win-MT\"" 
                "-l\"C:/Program Files/AMD/AOCL-Windows/amd-utils/lib/libaoclutils_static\""
            )
            set(BLIS_INCLUDE_DIR "C:/Program Files/AMD/AOCL-Windows/amd-blis/include/ILP64")
            set(FLAME_INCLUDE_DIR "C:/Program Files/AMD/AOCL-Windows/amd-libflame/include/ILP64")
            set(BLIS_DEFINES BLIS_ENABLE_NO_UNDERSCORE_API BLIS_ENABLE_CBLAS)
        else()
            find_library(BLAS_LIBRARY libopenblas PATHS ${OPENBLAS_DIR}/lib NO_DEFAULT_PATH)
            if(NOT BLAS_LIBRARY)
                find_package(OpenBLAS CONFIG REQUIRED)
                set(BLAS_LIBRARY OpenBLAS::OpenBLAS)
                set(BLAS_INCLUDE_DIR "")
            else()
                set(BLAS_INCLUDE_DIR ${OPENBLAS_DIR}/include CACHE PATH "OpenBLAS library include path")
            endif()
        endif()
    endif()
    
    # BLIS interface source if BLIS is being used
    if(DEFINED BLIS_INCLUDE_DIR)
        set(BLIS_CPP "${CMAKE_CURRENT_SOURCE_DIR}/common/blis_interface.cpp")
    endif()
    
    # Find GTest
    if(EXISTS "${BUILD_DIR}/deps/deps-install/lib/libgtest.a")
        set(GTEST_ROOT "${BUILD_DIR}/deps/deps-install")
    endif()
    find_package(GTest REQUIRED)
    
    message(STATUS "Linking Libs: ${BLAS_LIBRARY}")
    
    # Find hipblas if not already a target
    if(NOT TARGET hipblas)
        find_package(hipblas REQUIRED CONFIG)
    endif()
    
    # Build test client
    if(HIPBLAS_BUILD_TESTING)
        add_subdirectory(gtest)
    endif()
    
    # Build benchmark client
    if(HIPBLAS_ENABLE_BENCHMARKS)
        add_subdirectory(benchmarks)
    endif()
endif()

# Setup client common files
set(HIPBLAS_COMMON "${PROJECT_BINARY_DIR}/staging/hipblas_common.yaml")
add_custom_command(
    OUTPUT "${HIPBLAS_COMMON}"
    COMMAND ${CMAKE_COMMAND} -E copy 
        "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_common.yaml" 
        "${HIPBLAS_COMMON}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_common.yaml"
)

set(HIPBLAS_TEMPLATE "${PROJECT_BINARY_DIR}/staging/hipblas_template.yaml")
add_custom_command(
    OUTPUT "${HIPBLAS_TEMPLATE}"
    COMMAND ${CMAKE_COMMAND} -E copy 
        "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_template.yaml" 
        "${HIPBLAS_TEMPLATE}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_template.yaml"
)

set(HIPBLAS_SMOKE "${PROJECT_BINARY_DIR}/staging/hipblas_smoke.yaml")
add_custom_command(
    OUTPUT "${HIPBLAS_SMOKE}"
    COMMAND ${CMAKE_COMMAND} -E copy 
        "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_smoke.yaml" 
        "${HIPBLAS_SMOKE}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas_smoke.yaml"
)

set(HIPBLAS_GENTEST "${PROJECT_BINARY_DIR}/staging/hipblas_gentest.py")
add_custom_command(
    OUTPUT "${HIPBLAS_GENTEST}"
    COMMAND ${CMAKE_COMMAND} -E copy 
        "${CMAKE_CURRENT_SOURCE_DIR}/common/hipblas_gentest.py" 
        "${HIPBLAS_GENTEST}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/common/hipblas_gentest.py"
)

set(HIPBLAS_CLIENTS_README "${PROJECT_BINARY_DIR}/staging/hipblas_clients_readme.txt")
add_custom_command(
    OUTPUT "${HIPBLAS_CLIENTS_README}"
    COMMAND ${CMAKE_COMMAND} -E copy 
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_clients_readme.txt" 
        "${HIPBLAS_CLIENTS_README}"
    DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_clients_readme.txt"
)

add_custom_target(
    hipblas-clients-common 
    DEPENDS 
        "${HIPBLAS_COMMON}" 
        "${HIPBLAS_TEMPLATE}" 
        "${HIPBLAS_SMOKE}" 
        "${HIPBLAS_GENTEST}" 
        "${HIPBLAS_CLIENTS_README}"
)

# Install client common files
if(HIPBLAS_BUILD_TESTING OR HIPBLAS_ENABLE_BENCHMARKS)
    rocm_install(
        FILES 
            ${HIPBLAS_COMMON} 
            ${HIPBLAS_TEMPLATE} 
            ${HIPBLAS_SMOKE}
        DESTINATION "${CMAKE_INSTALL_BINDIR}"
        COMPONENT clients-common
    )
    rocm_install(
        PROGRAMS ${HIPBLAS_GENTEST}
        DESTINATION "${CMAKE_INSTALL_BINDIR}"
        COMPONENT clients-common
    )
endif()

if(HIPBLAS_ENABLE_CLIENT)
    # This readme prevents an empty package hipblas-clients
    rocm_install(
        FILES ${HIPBLAS_CLIENTS_README}
        DESTINATION "${CMAKE_INSTALL_BINDIR}"
        COMPONENT clients
    )
endif()
