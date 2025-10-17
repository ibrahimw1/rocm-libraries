# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

cmake_minimum_required(VERSION 3.20)

option(CMAKE_EXPORT_COMPILE_COMMANDS "Generate compile commands json file." ON)

# Determine if this is a subproject
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_SOURCE_DIR}")
    set(HIPBLAS_IS_SUBPROJECT FALSE)
else()
    set(HIPBLAS_IS_SUBPROJECT TRUE)
endif()

project(hipblas VERSION 3.1.0 LANGUAGES CXX)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include(dependencies)
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

rocm_setup_version(VERSION ${PROJECT_VERSION})
set(hipblas_SOVERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}")

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    if(WIN32)
        set(CMAKE_INSTALL_PREFIX "C:/hipSDK" CACHE PATH "Install path prefix." FORCE)
    else()
        set(CMAKE_INSTALL_PREFIX "/opt/rocm" CACHE PATH "Install path prefix." FORCE)
    endif()
endif()

# Build options
option(HIPBLAS_BUILD_SHARED_LIBS "Build hipBLAS as a shared library" ON)
option(HIPBLAS_ENABLE_SOLVER "Add functions from rocSOLVER" ON)
option(HIPBLAS_ENABLE_COVERAGE "Build with code coverage enabled" ON)
option(HIPBLAS_ENABLE_ASAN "Build with address sanitizer enabled" OFF)

# Client options
option(HIPBLAS_ENABLE_CLIENT "Build hipBLAS clients" ON)
if(HIPBLAS_ENABLE_CLIENT)
    option(HIPBLAS_BUILD_TESTING "Build tests" ON)
    option(HIPBLAS_ENABLE_BENCHMARKS "Build benchmarks" ON)
    option(HIPBLAS_ENABLE_SAMPLES "Build samples" ON)
    option(HIPBLAS_ENABLE_FORTRAN "Build Fortran clients" ON)
    option(HIPBLAS_ENABLE_BLIS "Use BLIS for reference BLAS" ON)
endif()

# Find dependencies
find_package(hip REQUIRED)

# Detect HIP platform
if(NOT DEFINED HIP_PLATFORM)
    if(DEFINED ENV{HIP_PLATFORM})
        set(HIP_PLATFORM $ENV{HIP_PLATFORM})
    elseif(hip_FOUND AND DEFINED HIP_PLATFORM)
        # Use HIP_PLATFORM from hip config
    else()
        set(HIP_PLATFORM "amd")
    endif()
endif()

find_package(hipblas-common REQUIRED)

if(HIP_PLATFORM STREQUAL "amd")
    find_package(rocblas REQUIRED)
    if(HIPBLAS_ENABLE_SOLVER)
        find_package(rocsolver REQUIRED)
    endif()
else()
    find_package(CUDA REQUIRED)
endif()

if(HIPBLAS_ENABLE_FORTRAN)
    enable_language(Fortran)
endif()

# Create main library target
if(HIPBLAS_BUILD_SHARED_LIBS)
    add_library(hipblas SHARED)
else()
    add_library(hipblas STATIC)
endif()
add_library(roc::hipblas ALIAS hipblas)
rocm_set_soversion(hipblas ${hipblas_SOVERSION})

target_compile_features(hipblas PUBLIC cxx_std_17)

set_target_properties(hipblas PROPERTIES
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN ON
)

target_include_directories(hipblas
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/library/include>
        $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include/hipblas>
        $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Link dependencies
target_link_libraries(hipblas PUBLIC roc::hipblas-common)

if(HIP_PLATFORM STREQUAL "amd")
    target_link_libraries(hipblas
        PRIVATE
            roc::rocblas
            hip::host
    )
    if(HIPBLAS_ENABLE_SOLVER)
        target_link_libraries(hipblas PRIVATE roc::rocsolver)
        target_compile_definitions(hipblas PUBLIC __HIP_PLATFORM_SOLVER__)
    endif()
else()
    target_compile_definitions(hipblas PRIVATE
        __HIP_PLATFORM_NVCC__
        __HIP_PLATFORM_NVIDIA__
    )
    target_link_libraries(hipblas PRIVATE ${CUDA_CUBLAS_LIBRARIES})
endif()

if(HIPBLAS_ENABLE_ASAN)
    # Address sanitizer flags - needed for specific debugging scenarios
    target_compile_options(hipblas PRIVATE -fsanitize=address -shared-libasan)
    target_link_options(hipblas PRIVATE -fsanitize=address -shared-libasan -fuse-ld=lld)
endif()

# Add library subdirectory to populate target
add_subdirectory(library)

# Client library and executables
if(HIPBLAS_ENABLE_CLIENT)
    # Common client library
    add_library(hipblas-clients-common OBJECT)
    
    # Client executables
    if(HIPBLAS_BUILD_TESTING)
        add_executable(hipblas-test)
    endif()
    
    if(HIPBLAS_ENABLE_BENCHMARKS)
        add_executable(hipblas-bench)
    endif()
    
    # Sample executables - defined explicitly in samples/CMakeLists.txt
    
    add_subdirectory(clients)
endif()

# Documentation
if(HIPBLAS_ENABLE_DOCS)
    add_subdirectory(docs)
endif()

# Installation
rocm_install_targets(
    TARGETS hipblas
    INCLUDE ${CMAKE_BINARY_DIR}/include
)

# Export targets
if(HIP_PLATFORM STREQUAL "amd")
    rocm_export_targets(
        TARGETS roc::hipblas
        DEPENDS PACKAGE hip
        DEPENDS PACKAGE hipblas-common
        NAMESPACE roc::
    )
else()
    rocm_export_targets(
        TARGETS roc::hipblas
        DEPENDS PACKAGE HIP
        DEPENDS PACKAGE hipblas-common
        NAMESPACE roc::
    )
endif()

# Install Fortran module if enabled
if(HIPBLAS_ENABLE_FORTRAN)
    rocm_install(
        FILES "${CMAKE_CURRENT_SOURCE_DIR}/library/src/hipblas_module.f90"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/hipblas"
    )
endif()

# Client installations
if(HIPBLAS_ENABLE_CLIENT)
    # Common files
    set(HIPBLAS_COMMON "${PROJECT_BINARY_DIR}/staging/hipblas_common.yaml")
    set(HIPBLAS_TEMPLATE "${PROJECT_BINARY_DIR}/staging/hipblas_template.yaml")
    set(HIPBLAS_SMOKE "${PROJECT_BINARY_DIR}/staging/hipblas_smoke.yaml")
    set(HIPBLAS_GENTEST "${PROJECT_BINARY_DIR}/staging/hipblas_gentest.py")
    set(HIPBLAS_CLIENTS_README "${PROJECT_BINARY_DIR}/staging/hipblas_clients_readme.txt")
    
    if(HIPBLAS_BUILD_TESTING OR HIPBLAS_ENABLE_BENCHMARKS)
        rocm_install(
            FILES ${HIPBLAS_COMMON} ${HIPBLAS_TEMPLATE} ${HIPBLAS_SMOKE}
            DESTINATION "${CMAKE_INSTALL_BINDIR}"
            COMPONENT clients-common
        )
        rocm_install(
            PROGRAMS ${HIPBLAS_GENTEST}
            DESTINATION "${CMAKE_INSTALL_BINDIR}"
            COMPONENT clients-common
        )
        rocm_install(
            FILES ${HIPBLAS_CLIENTS_README}
            DESTINATION "${CMAKE_INSTALL_BINDIR}"
            COMPONENT clients
        )
    endif()
    
    # Install test executable and data
    if(HIPBLAS_BUILD_TESTING)
        rocm_install(TARGETS hipblas-test COMPONENT tests)
        rocm_install(
            FILES "${PROJECT_BINARY_DIR}/staging/hipblas_gtest.data"
            DESTINATION "${CMAKE_INSTALL_BINDIR}"
            COMPONENT tests
        )
    endif()
    
    # Install benchmark executable
    if(HIPBLAS_ENABLE_BENCHMARKS)
        rocm_install(TARGETS hipblas-bench COMPONENT benchmarks)
    endif()
    
    # Sample installations handled in samples/CMakeLists.txt
endif()

# Package configuration
if(NOT HIPBLAS_IS_SUBPROJECT)
    if(WIN32)
        set(CPACK_SOURCE_GENERATOR "ZIP")
        set(CPACK_GENERATOR "ZIP")
    endif()
    
    set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md")
    set(CPACK_RPM_PACKAGE_LICENSE "MIT")
    
    if(WIN32)
        set(CPACK_SET_DESTDIR FALSE)
        set(CPACK_PACKAGE_INSTALL_DIRECTORY "C:/hipSDK")
        set(CPACK_PACKAGING_INSTALL_PREFIX "")
        set(CPACK_INCLUDE_TOPLEVEL_DIRECTORY OFF)
    else()
        if(NOT CPACK_PACKAGING_INSTALL_PREFIX)
            set(CPACK_PACKAGING_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}")
        endif()
    endif()
    
    set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "\${CPACK_PACKAGING_INSTALL_PREFIX}")
    
    # Platform-specific package dependencies
    if(HIP_PLATFORM STREQUAL "amd")
        set(rocblas_minimum 5.1.0)
        set(rocsolver_minimum 3.31.0)
        rocm_package_add_dependencies(
            SHARED_DEPENDS "rocblas >= ${rocblas_minimum}" "rocsolver >= ${rocsolver_minimum}"
        )
        rocm_package_add_rpm_dependencies(
            STATIC_DEPENDS "rocblas-static-devel >= ${rocblas_minimum}" "rocsolver-static-devel >= ${rocsolver_minimum}"
        )
        rocm_package_add_deb_dependencies(
            STATIC_DEPENDS "rocblas-static-dev >= ${rocblas_minimum}" "rocsolver-static-dev >= ${rocsolver_minimum}"
        )
    endif()
    
    set(hipblas_common_minimum 1.3.0)
    if(HIPBLAS_BUILD_SHARED_LIBS)
        rocm_package_add_deb_dependencies(COMPONENT devel DEPENDS "hipblas-common-dev >= ${hipblas_common_minimum}")
        rocm_package_add_rpm_dependencies(COMPONENT devel DEPENDS "hipblas-common-devel >= ${hipblas_common_minimum}")
    else()
        rocm_package_add_deb_dependencies(COMPONENT devel DEPENDS "hipblas-common-static-dev >= ${hipblas_common_minimum}")
        rocm_package_add_rpm_dependencies(COMPONENT devel DEPENDS "hipblas-common-static-devel >= ${hipblas_common_minimum}")
    endif()
    
    # Client package components
    if(HIPBLAS_ENABLE_CLIENT)
        rocm_package_setup_component(clients)
        rocm_package_setup_client_component(clients-common)
        
        if(HIPBLAS_BUILD_TESTING)
            rocm_package_setup_client_component(tests DEPENDS COMPONENT clients-common)
        endif()
        
        if(HIPBLAS_ENABLE_BENCHMARKS)
            rocm_package_setup_client_component(benchmarks DEPENDS COMPONENT clients-common)
        endif()
        
        if(HIPBLAS_ENABLE_SAMPLES)
            rocm_package_setup_client_component(samples DEPENDS COMPONENT clients-common)
        endif()
        
        # Fortran dependencies for client components
        if(HIPBLAS_ENABLE_FORTRAN)
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
            
            if(HIPBLAS_BUILD_TESTING)
                rocm_package_add_rpm_dependencies(COMPONENT tests DEPENDS "${GFORTRAN_RPM}")
                rocm_package_add_deb_dependencies(COMPONENT tests DEPENDS "${GFORTRAN_DEB}")
            endif()
            if(HIPBLAS_ENABLE_BENCHMARKS)
                rocm_package_add_rpm_dependencies(COMPONENT benchmarks DEPENDS "${GFORTRAN_RPM}")
                rocm_package_add_deb_dependencies(COMPONENT benchmarks DEPENDS "${GFORTRAN_DEB}")
            endif()
            if(HIPBLAS_ENABLE_SAMPLES)
                rocm_package_add_rpm_dependencies(COMPONENT samples DEPENDS "${GFORTRAN_RPM}")
                rocm_package_add_deb_dependencies(COMPONENT samples DEPENDS "${GFORTRAN_DEB}")
            endif()
        endif()
    endif()
    
    # Package name
    if(HIP_PLATFORM STREQUAL "amd")
        set(package_name hipblas)
    else()
        set(package_name hipblas-alt)
    endif()
    
    set(HIPBLAS_CONFIG_DIR "\${CPACK_PACKAGING_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}"
        CACHE PATH "Path placed into ldconfig file"
    )
    
    rocm_create_package(
        NAME ${package_name}
        DESCRIPTION "ROCm BLAS marshalling library"
        MAINTAINER "hipBLAS Maintainer <hipblas-maintainer@amd.com>"
        LDCONFIG
        LDCONFIG_DIR ${HIPBLAS_CONFIG_DIR}
    )
endif()

# Code coverage targets
if(HIPBLAS_ENABLE_COVERAGE)
    find_program(LLVM_PROFDATA llvm-profdata REQUIRED HINTS ${ROCM_PATH}/llvm/bin PATHS /opt/rocm/llvm/bin)
    find_program(LLVM_COV llvm-cov REQUIRED HINTS ${ROCM_PATH}/llvm/bin PATHS /opt/rocm/llvm/bin)
    
    # Coverage test executable name
    set(coverage_test ./clients/staging/hipblas-test)
    if(CMAKE_BUILD_TYPE STREQUAL "Debug")
        set(coverage_test ./clients/staging/hipblas-test-d)
    endif()
    
    add_custom_target(coverage_analysis
        COMMAND echo Coverage GTEST_FILTER=\${GTEST_FILTER}
        COMMAND ${CMAKE_COMMAND} -E make_directory ./coverage/profraw
        COMMAND ${CMAKE_COMMAND} -E env
            LLVM_PROFILE_FILE="./coverage-report/profraw/hipblas-coverage_%m.profraw"
            ${coverage_test} --gtest_filter=\"\${GTEST_FILTER}\"
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
    
    add_dependencies(coverage_analysis hipblas)
    
    add_custom_target(coverage
        DEPENDS coverage_analysis
        COMMAND ${LLVM_PROFDATA} merge -sparse
            ./coverage-report/profraw/hipblas-coverage_*.profraw
            -o ./coverage-report/hipblas.profdata
        COMMAND ${LLVM_COV} report
            -object ./library/src/libhipblas.so
            -instr-profile=./coverage-report/hipblas.profdata
        COMMAND ${LLVM_COV} show
            -object ./library/src/libhipblas.so
            -instr-profile=./coverage-report/hipblas.profdata
            -format=html
            -output-dir=coverage-report
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
    
    add_custom_target(coverage_cleanup
        COMMAND find ${CMAKE_BINARY_DIR} -name *.gcda -delete
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
endif()