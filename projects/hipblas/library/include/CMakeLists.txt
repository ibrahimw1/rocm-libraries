# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

# Install public headers
rocm_install(
    FILES
        "${CMAKE_CURRENT_SOURCE_DIR}/hipblas.h"
        "${PROJECT_BINARY_DIR}/include/hipblas/hipblas-version.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/hipblas"
    COMPONENT devel
)

# Install Fortran module if not on Windows
if(NOT WIN32)
    rocm_install(
        FILES
            "${PROJECT_BINARY_DIR}/include/hipblas_module.mod"
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/hipblas"
        COMPONENT devel
    )
endif()