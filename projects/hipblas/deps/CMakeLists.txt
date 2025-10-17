# Copyright Advanced Micro Devices, Inc., or its affiliates.
# SPDX-License-Identifier: MIT

cmake_minimum_required(VERSION 3.20)
project(hipblas_deps_dummy LANGUAGES NONE)

# Transitional no-op build to satisfy legacy build system
# Dependencies are now managed via FetchContent in the main build
message(STATUS "hipBLAS deps: transitional no-op build. Dependencies managed in main CMakeLists.txt.")

add_custom_target(dummy_all ALL
    COMMENT "No-op: dependencies managed in main build via FetchContent"
)

# Provide successful no-op install
install(CODE [[
    message(STATUS "hipBLAS deps: no-op install. Dependencies installed via main build.")
]])
