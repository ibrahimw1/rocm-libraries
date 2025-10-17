# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

# Generate version header
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas-version.h.in"
    "${PROJECT_BINARY_DIR}/include/hipblas/hipblas-version.h"
)

# Copy public header to build directory
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/include/hipblas.h"
    "${PROJECT_BINARY_DIR}/include/hipblas/hipblas.h"
    COPYONLY
)

# Generate export header for symbol visibility
include(GenerateExportHeader)
generate_export_header(hipblas
    EXPORT_FILE_NAME "${PROJECT_BINARY_DIR}/include/hipblas/hipblas-export.h"
)

# Set Fortran module output directory if Fortran is enabled
if(HIPBLAS_ENABLE_FORTRAN)
    set(CMAKE_Fortran_MODULE_DIRECTORY "${PROJECT_BINARY_DIR}/include")
endif()

# Add sources subdirectory
add_subdirectory(src)
