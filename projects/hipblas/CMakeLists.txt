# Copyright (C) 2016-2025 Advanced Micro Devices, Inc. All rights reserved.
# SPDX-License-Identifier: MIT

cmake_minimum_required(VERSION 3.20)
project(hipblas VERSION 3.1.0 LANGUAGES CXX)

if(POLICY CMP0135)
    cmake_policy(SET CMP0135 NEW) # FetchContent timestamp handling
endif()

if(POLICY CMP0077)
    cmake_policy(SET CMP0077 NEW) # option() honors normal variables
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Set cpack license file here to avoid rocm-cmake warning
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md")

include(dependencies)
include(CMakeDependentOption)
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

rocm_setup_version(VERSION ${PROJECT_VERSION})
set(hipblas_SOVERSION 3.1)

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    if(WIN32)
        set(CMAKE_INSTALL_PREFIX "C:/hipSDK" CACHE PATH "Install path prefix." FORCE)
    else()
        set(CMAKE_INSTALL_PREFIX "/opt/rocm" CACHE PATH "Install path prefix." FORCE)
    endif()
endif()

# --- Build options and validation ---

option(CMAKE_EXPORT_COMPILE_COMMANDS "Export compile_commands.json for clang tooling support" ON)

# Backend selection
option(HIPBLAS_ENABLE_CUDA "Build hipBLAS with CUDA backend." OFF)
cmake_dependent_option(
    HIPBLAS_ENABLE_HIP "Build hipBLAS with HIP backend." ON "NOT HIPBLAS_ENABLE_CUDA"
    OFF
)
if(HIPBLAS_ENABLE_CUDA AND HIPBLAS_ENABLE_HIP)
    message(STATUS "HIPBLAS_ENABLE_CUDA: ${HIPBLAS_ENABLE_CUDA}")
    message(STATUS "HIPBLAS_ENABLE_HIP: ${HIPBLAS_ENABLE_HIP}")
    message(FATAL_ERROR "Cannot enable CUDA and HIP support simultaneously.")
endif()

# Library options
option(HIPBLAS_BUILD_SHARED_LIBS "Build the hipBLAS shared library." ON)
option(HIPBLAS_ENABLE_SOLVER "Enable rocSOLVER functions in hipBLAS." ON)

# Testing and coverage
option(HIPBLAS_BUILD_TESTING "Build test client." ON)
option(HIPBLAS_BUILD_COVERAGE "Build tests with coverage enabled." OFF)

# Client options
option(HIPBLAS_ENABLE_CLIENT "Build hipBLAS clients." ON)
if(HIPBLAS_ENABLE_CLIENT)
    option(HIPBLAS_ENABLE_BENCHMARKS "Build benchmark client." ON)
    option(HIPBLAS_ENABLE_SAMPLES "Build client samples." ON)
    option(HIPBLAS_ENABLE_FORTRAN "Build Fortran clients." ON)
endif()

# Additional build options
option(HIPBLAS_ENABLE_ASAN "Build with address sanitizer enabled." OFF)
option(HIPBLAS_ENABLE_VERBOSE "Output additional build information." OFF)
option(HIPBLAS_BUILD_DOCS "Build documentation." OFF)

# Deprecated option handling
if(DEFINED USE_CUDA)
    message(DEPRECATION "USE_CUDA is deprecated. Use HIPBLAS_ENABLE_CUDA instead.")
    if(USE_CUDA)
        set(HIPBLAS_ENABLE_CUDA ON CACHE BOOL "Build with CUDA backend" FORCE)
        set(HIPBLAS_ENABLE_HIP OFF CACHE BOOL "Build with HIP backend" FORCE)
    endif()
endif()

# Backward compatibility for old option names
if(DEFINED BUILD_WITH_SOLVER)
    message(DEPRECATION "BUILD_WITH_SOLVER is deprecated. Use HIPBLAS_ENABLE_SOLVER instead.")
    set(HIPBLAS_ENABLE_SOLVER ${BUILD_WITH_SOLVER} CACHE BOOL "Enable rocSOLVER functions" FORCE)
endif()

if(DEFINED BUILD_SHARED_LIBS AND NOT DEFINED HIPBLAS_BUILD_SHARED_LIBS)
    set(HIPBLAS_BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS})
endif()

# Validate build configuration
if(HIPBLAS_BUILD_COVERAGE)
    if(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
        message(FATAL_ERROR "Code coverage is only supported for CMAKE_BUILD_TYPE=Debug.")
    endif()
endif()

# Setup Fortran if needed
if(HIPBLAS_ENABLE_CLIENT AND HIPBLAS_ENABLE_FORTRAN)
    if(NOT DEFINED CMAKE_Fortran_COMPILER AND NOT DEFINED ENV{FC})
        set(CMAKE_Fortran_COMPILER "gfortran")
    endif()
    enable_language(Fortran)
endif()

# --- Dependency Setup ---

find_package(Python3 REQUIRED COMPONENTS Interpreter)
if(NOT python)
    set(python "${Python3_EXECUTABLE}")
endif()

find_package(hip REQUIRED)

if(HIPBLAS_ENABLE_CUDA)
    find_package(CUDAToolkit REQUIRED)
    find_package(CUDA REQUIRED)
endif()

if(HIPBLAS_ENABLE_HIP)
    find_package(hipblas-common REQUIRED)
endif()

# --- Define library target ---

if(HIPBLAS_BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
    add_library(hipblas SHARED)
else()
    add_library(hipblas STATIC)
endif()

add_library(roc::hipblas ALIAS hipblas)
rocm_set_soversion(hipblas ${hipblas_SOVERSION})

target_compile_features(hipblas PRIVATE cxx_std_17)

# Platform-specific compile definitions
if(HIPBLAS_ENABLE_CUDA)
    target_compile_definitions(hipblas PRIVATE __HIP_PLATFORM_NVCC__ __HIP_PLATFORM_NVIDIA__)
endif()

if(HIPBLAS_ENABLE_HIP)
    target_compile_definitions(hipblas PRIVATE __HIP_PLATFORM_AMD__)
    if(HIPBLAS_ENABLE_SOLVER)
        target_compile_definitions(hipblas PRIVATE __HIP_PLATFORM_SOLVER__)
    endif()
endif()

# Address sanitizer support
if(HIPBLAS_ENABLE_ASAN)
    target_compile_options(hipblas PRIVATE -fsanitize=address -shared-libasan)
    target_link_options(hipblas PRIVATE -fsanitize=address -shared-libasan -fuse-ld=lld)
endif()

# Coverage support
if(HIPBLAS_BUILD_COVERAGE)
    target_compile_options(hipblas PRIVATE -g -O0 -fprofile-instr-generate -fcoverage-mapping)
    target_link_options(hipblas PUBLIC -fprofile-instr-generate)
endif()

# Symbol visibility
set_target_properties(
    hipblas PROPERTIES 
    CXX_EXTENSIONS OFF
    CXX_VISIBILITY_PRESET "hidden" 
    VISIBILITY_INLINES_HIDDEN ON
)

# Build library from subdirectories
add_subdirectory(library)

# Build clients
if(HIPBLAS_ENABLE_CLIENT)
    add_subdirectory(clients)
endif()

# Build docs
if(HIPBLAS_BUILD_DOCS)
    add_subdirectory(docs)
endif()

# --- Packaging and installation ---

if(HIPBLAS_ENABLE_CLIENT)
    include(ROCmClientPackages)
    rocm_package_setup_component(clients)
    rocm_package_setup_client_component(clients-common)
    
    # Determine gfortran package names based on OS
    if(NOT CLIENTS_OS)
        rocm_set_os_id(CLIENTS_OS)
        string(TOLOWER "${CLIENTS_OS}" CLIENTS_OS)
        rocm_read_os_release(CLIENTS_OS_VERSION VERSION_ID)
    endif()
    
    set(GFORTRAN_RPM "libgfortran4")
    set(GFORTRAN_DEB "libgfortran4")
    if(CLIENTS_OS STREQUAL "centos" OR CLIENTS_OS STREQUAL "rhel" OR CLIENTS_OS STREQUAL "almalinux")
        if(CLIENTS_OS_VERSION VERSION_GREATER_EQUAL "8")
            set(GFORTRAN_RPM "libgfortran")
        endif()
    elseif(CLIENTS_OS STREQUAL "ubuntu" AND CLIENTS_OS_VERSION VERSION_GREATER_EQUAL "20.04")
        set(GFORTRAN_DEB "libgfortran5")
    elseif(CLIENTS_OS STREQUAL "mariner" OR CLIENTS_OS STREQUAL "azurelinux")
        set(GFORTRAN_RPM "gfortran")
    endif()
endif()

if(HIPBLAS_BUILD_TESTING OR BUILD_TESTING)
    rocm_package_setup_client_component(
        tests
        DEPENDS
        COMPONENT clients-common
        DEB "${GFORTRAN_DEB}"
        RPM "${GFORTRAN_RPM}"
    )
endif()

if(HIPBLAS_ENABLE_BENCHMARKS)
    rocm_package_setup_client_component(
        benchmarks
        DEPENDS
        COMPONENT clients-common
        DEB "${GFORTRAN_DEB}"
        RPM "${GFORTRAN_RPM}"
    )
endif()

if(HIPBLAS_ENABLE_SAMPLES)
    rocm_package_setup_client_component(
        samples
        DEPENDS
        COMPONENT clients-common
        DEB "${GFORTRAN_DEB}"
        RPM "${GFORTRAN_RPM}"
    )
endif()

# Install targets
rocm_install(
    TARGETS hipblas
    EXPORT hipblas-targets
    ARCHIVE COMPONENT "devel"
    LIBRARY COMPONENT "runtime"
    RUNTIME COMPONENT "runtime"
)

rocm_install(
    DIRECTORY "${PROJECT_BINARY_DIR}/include/" 
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    COMPONENT "devel"
)

# Export targets
if(HIPBLAS_ENABLE_HIP)
    set(static_depends)
    if(NOT HIPBLAS_BUILD_SHARED_LIBS)
        list(APPEND static_depends PACKAGE rocblas)
        if(HIPBLAS_ENABLE_SOLVER)
            list(APPEND static_depends PACKAGE rocsolver)
        endif()
    endif()
    
    rocm_export_targets(
        NAMESPACE roc::
        TARGETS hipblas
        EXPORT hipblas-targets
        DEPENDS PACKAGE hip
        DEPENDS PACKAGE hipblas-common
        STATIC_DEPENDS ${static_depends}
    )
else()
    rocm_export_targets(
        NAMESPACE roc::
        TARGETS hipblas
        EXPORT hipblas-targets
        DEPENDS PACKAGE HIP
        DEPENDS PACKAGE hipblas-common
    )
endif()

# CPack configuration
if(WIN32)
    set(CPACK_SOURCE_GENERATOR "ZIP")
    set(CPACK_GENERATOR "ZIP")
    set(CPACK_SET_DESTDIR FALSE)
    set(CPACK_INCLUDE_TOPLEVEL_DIRECTORY OFF)
elseif(NOT CPACK_PACKAGING_INSTALL_PREFIX)
    set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
endif()

# Package dependencies
if(HIPBLAS_ENABLE_HIP)
    set(rocblas_minimum 5.1.0)
    set(rocsolver_minimum 3.31.0)
    rocm_package_add_dependencies(SHARED_DEPENDS "rocblas >= ${rocblas_minimum}")
    if(HIPBLAS_ENABLE_SOLVER)
        rocm_package_add_dependencies(SHARED_DEPENDS "rocsolver >= ${rocsolver_minimum}")
    endif()
    rocm_package_add_rpm_dependencies(STATIC_DEPENDS "rocblas-static-devel >= ${rocblas_minimum}")
    rocm_package_add_deb_dependencies(STATIC_DEPENDS "rocblas-static-dev >= ${rocblas_minimum}")
    if(HIPBLAS_ENABLE_SOLVER)
        rocm_package_add_rpm_dependencies(STATIC_DEPENDS "rocsolver-static-devel >= ${rocsolver_minimum}")
        rocm_package_add_deb_dependencies(STATIC_DEPENDS "rocsolver-static-dev >= ${rocsolver_minimum}")
    endif()
endif()

set(hipblas_common_minimum 1.3.0)
if(HIPBLAS_BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
    rocm_package_add_deb_dependencies(COMPONENT devel DEPENDS "hipblas-common-dev >= ${hipblas_common_minimum}")
    rocm_package_add_rpm_dependencies(COMPONENT devel DEPENDS "hipblas-common-devel >= ${hipblas_common_minimum}")
else()
    rocm_package_add_deb_dependencies(COMPONENT devel DEPENDS "hipblas-common-static-dev >= ${hipblas_common_minimum}")
    rocm_package_add_rpm_dependencies(COMPONENT devel DEPENDS "hipblas-common-static-devel >= ${hipblas_common_minimum}")
endif()

set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "\${CPACK_PACKAGING_INSTALL_PREFIX}")

# Give hipblas compiled for CUDA backend a different name
if(HIPBLAS_ENABLE_HIP)
    set(package_name hipblas)
else()
    set(package_name hipblas-alt)
endif()

set(HIPBLAS_CONFIG_DIR "\${CPACK_PACKAGING_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}" CACHE PATH "Path placed into ldconfig file")

rocm_create_package(
    NAME ${package_name}
    DESCRIPTION "ROCm BLAS marshalling library"
    MAINTAINER "hipBLAS Maintainer <hipblas-maintainer@amd.com>"
    LDCONFIG
    LDCONFIG_DIR ${HIPBLAS_CONFIG_DIR}
)

# --- Coverage setup ---

if(HIPBLAS_BUILD_COVERAGE)
    message(STATUS "Configuring code coverage target")
    
    set(_coverage_dir "${CMAKE_CURRENT_BINARY_DIR}/coverage-report")
    set(_coverage_test $<TARGET_FILE:hipblas-test>)
    
    add_custom_target(
        code_cov_tests
        DEPENDS hipblas-test
        COMMAND ${CMAKE_COMMAND} -E rm -rf "${_coverage_dir}"
        COMMAND ${CMAKE_COMMAND} -E make_directory "${_coverage_dir}/profraw"
        COMMAND
            ${CMAKE_COMMAND} -E env
            LLVM_PROFILE_FILE="${_coverage_dir}/profraw/hipblas-coverage_%p.profraw"
            ${_coverage_test} --gtest_filter=\"\${GTEST_FILTER}\"
        COMMENT "Running coverage tests"
    )
    
    find_program(LLVM_PROFDATA llvm-profdata REQUIRED HINTS ${ROCM_PATH}/llvm/bin)
    find_program(LLVM_COV llvm-cov REQUIRED HINTS ${ROCM_PATH}/llvm/bin)
    
    add_custom_target(
        coverage
        DEPENDS code_cov_tests
        COMMAND
            ${LLVM_PROFDATA} merge -sparse "${_coverage_dir}/profraw/hipblas-coverage_*.profraw"
            -o "${_coverage_dir}/hipblas.profdata"
        COMMAND ${LLVM_COV} report -object $<TARGET_FILE:hipblas>
                -instr-profile="${_coverage_dir}/hipblas.profdata"
        COMMAND
            ${LLVM_COV} show -object $<TARGET_FILE:hipblas>
            -instr-profile="${_coverage_dir}/hipblas.profdata" -format=html
            -output-dir="${_coverage_dir}"
        COMMENT "Generating coverage... report can be found in ${_coverage_dir}/index.html"
    )
    
    add_custom_target(
        coverage_cleanup
        COMMAND ${CMAKE_COMMAND} -E rm -rf "${_coverage_dir}"
        COMMENT "Cleaning coverage files"
    )
endif()
