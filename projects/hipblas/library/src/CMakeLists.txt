# Copyright (C) 2016-2025 Advanced Micro Devices, Inc. All rights reserved.
# SPDX-License-Identifier: MIT

# Add source files to the library target
target_sources(
    hipblas
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_auxiliary.cpp"
)

# Add backend-specific sources
if(HIPBLAS_ENABLE_HIP)
    target_sources(
        hipblas 
        PRIVATE 
            "${CMAKE_CURRENT_SOURCE_DIR}/amd_detail/hipblas.cpp"
    )
    
    # Add verbose compilation flag for QC purposes
    message(STATUS "** NOTE: amd_detail/hipblas.cpp is compiled with the verbose flag -v for QC purposes.")
    set_source_files_properties(
        "${CMAKE_CURRENT_SOURCE_DIR}/amd_detail/hipblas.cpp"
        PROPERTIES COMPILE_FLAGS "-v"
    )
elseif(HIPBLAS_ENABLE_CUDA)
    target_sources(
        hipblas 
        PRIVATE 
            "${CMAKE_CURRENT_SOURCE_DIR}/nvidia_detail/hipblas.cpp"
    )
endif()

# Internal header includes
target_include_directories(
    hipblas
    PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}/include"
        "${CMAKE_CURRENT_SOURCE_DIR}"
)

# Platform-specific configuration
if(HIPBLAS_ENABLE_HIP)
    # Find rocBLAS
    if(NOT TARGET rocblas)
        if(CUSTOM_ROCBLAS)
            set(ENV{rocblas_DIR} ${CUSTOM_ROCBLAS})
            find_package(rocblas REQUIRED CONFIG NO_CMAKE_PATH)
        elseif(WIN32)
            find_package(rocblas REQUIRED CONFIG)
        else()
            find_package(rocblas REQUIRED CONFIG)
        endif()
    endif()
    
    target_link_libraries(hipblas PRIVATE roc::rocblas)
    target_link_libraries(hipblas PUBLIC hip::host)
    
    # Find rocSOLVER if enabled
    if(HIPBLAS_ENABLE_SOLVER)
        if(NOT TARGET rocsolver)
            if(CUSTOM_ROCSOLVER)
                set(ENV{rocsolver_DIR} ${CUSTOM_ROCSOLVER})
                find_package(rocsolver REQUIRED CONFIG NO_CMAKE_PATH)
                
                # Custom rocsolver include directories take precedence
                target_include_directories(
                    hipblas
                    SYSTEM PRIVATE 
                        $<BUILD_INTERFACE:${ROCSOLVER_INCLUDE_DIRS}>
                )
            elseif(WIN32)
                find_package(rocsolver REQUIRED CONFIG)
            else()
                find_package(rocsolver REQUIRED CONFIG)
            endif()
        endif()
        
        target_link_libraries(hipblas PRIVATE roc::rocsolver)
    endif()
    
    # Custom target support
    if(CUSTOM_TARGET)
        target_link_libraries(hipblas PRIVATE hip::${CUSTOM_TARGET})
    endif()
    
    # External header includes as system files
    target_include_directories(
        hipblas
        SYSTEM PRIVATE
            $<BUILD_INTERFACE:${HIP_INCLUDE_DIRS}>
            $<BUILD_INTERFACE:${ROCBLAS_INCLUDE_DIRS}>
            $<BUILD_INTERFACE:${ROCSOLVER_INCLUDE_DIRS}>
    )
    
elseif(HIPBLAS_ENABLE_CUDA)
    # CUDA backend
    target_link_libraries(hipblas PRIVATE CUDA::cublas CUDA::cudart)
    
    # External header includes as system files
    target_include_directories(
        hipblas
        SYSTEM PRIVATE
            $<BUILD_INTERFACE:${CUDAToolkit_INCLUDE_DIRS}>
            $<BUILD_INTERFACE:${CUDA_INCLUDE_DIRS}>
    )
endif()

# Windows-specific post-build commands
if(WIN32)
    # Copy DLL to clients staging directory
    add_custom_command(
        TARGET hipblas POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy 
            $<TARGET_FILE:hipblas>
            "${PROJECT_BINARY_DIR}/clients/staging/$<TARGET_FILE_NAME:hipblas>"
        COMMENT "Copying hipblas DLL to clients staging directory"
    )
    
    # Copy PDB file for Debug builds
    add_custom_command(
        TARGET hipblas POST_BUILD
        COMMAND $<$<CONFIG:Debug>:${CMAKE_COMMAND}> -E copy 
            "${PROJECT_BINARY_DIR}/staging/hipblas.pdb"
            "${PROJECT_BINARY_DIR}/clients/staging/hipblas.pdb"
        COMMENT "Copying hipblas PDB to clients staging directory"
    )
endif()

# Install Fortran module file
if(HIPBLAS_ENABLE_FORTRAN)
    rocm_install(
        FILES "${CMAKE_CURRENT_SOURCE_DIR}/hipblas_module.f90"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/hipblas"
        COMPONENT devel
    )
endif()
