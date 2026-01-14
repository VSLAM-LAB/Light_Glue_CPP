cmake_minimum_required(VERSION 3.26)
project(LightGlue
        VERSION 1.0.0
        DESCRIPTION "C++ implementation of LightGlue"
        LANGUAGES CUDA CXX)

# Set default build type if not specified
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build." FORCE)
endif()
set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")

# Enable LTO/IPO only for Release builds
include(CheckIPOSupported)
check_ipo_supported(RESULT IPO_SUPPORTED OUTPUT IPO_ERROR)
if(IPO_SUPPORTED AND CMAKE_BUILD_TYPE STREQUAL "Release")
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
endif()

# Core configuration
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_ARCHITECTURES native)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Configure paths
set(LIGHTGLUE_MODELS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/models" CACHE PATH "Path to model weights directory")

# Find dependencies
set(LIBTORCH_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/libtorch")
list(PREPEND CMAKE_PREFIX_PATH "${LIBTORCH_DIR}")
#set(CMAKE_PREFIX_PATH ${LIBTORCH_DIR})

# --- CUDA discovery hints for pixi/conda (Torch/Caffe2 legacy FindCUDA) ---
if(DEFINED ENV{CONDA_PREFIX})
    message(STATUS "CONDA_PREFIX=$ENV{CONDA_PREFIX}")

    # Make CMake able to find packages from the pixi env as well
    list(PREPEND CMAKE_PREFIX_PATH "$ENV{CONDA_PREFIX}")

    # Conda/pixi CUDA headers+libs usually live under targets/x86_64-linux
    set(_CUDA_ROOT "$ENV{CONDA_PREFIX}")
    if(EXISTS "$ENV{CONDA_PREFIX}/targets/x86_64-linux/include/cuda_runtime.h")
        set(_CUDA_ROOT "$ENV{CONDA_PREFIX}/targets/x86_64-linux")
    endif()
    message(STATUS "Using CUDA root for legacy FindCUDA: ${_CUDA_ROOT}")

    # Modern FindCUDAToolkit hint (used by find_package(CUDAToolkit))
    set(CUDAToolkit_ROOT "$ENV{CONDA_PREFIX}" CACHE PATH "" FORCE)

    # Legacy FindCUDA hints (used by Torch/Caffe2)
    set(CUDA_TOOLKIT_ROOT_DIR "${_CUDA_ROOT}" CACHE PATH "" FORCE)
    set(CUDA_INCLUDE_DIRS "${_CUDA_ROOT}/include" CACHE PATH "" FORCE)

    # Libraries
    if(EXISTS "${_CUDA_ROOT}/lib/libcudart.so")
        set(CUDA_CUDART_LIBRARY "${_CUDA_ROOT}/lib/libcudart.so" CACHE FILEPATH "" FORCE)
    elseif(EXISTS "${_CUDA_ROOT}/lib64/libcudart.so")
        set(CUDA_CUDART_LIBRARY "${_CUDA_ROOT}/lib64/libcudart.so" CACHE FILEPATH "" FORCE)
    endif()
endif()


find_package(CUDAToolkit REQUIRED)  
find_package(Torch REQUIRED)
find_package(OpenCV REQUIRED)

# Check CUDA version
if(CUDAToolkit_VERSION VERSION_LESS "12.1")
    message(FATAL_ERROR "This project requires CUDA 12.1 or higher (found: ${CUDAToolkit_VERSION})")
endif()

# Debug flags for different compilers - removed problematic flags
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    set(DEBUG_FLAGS
            -g
            -Wall
            -Wextra
            -fno-omit-frame-pointer
    )
endif()

# Release flags
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    set(RELEASE_FLAGS
            -O3
            -march=native
            -mtune=native
            -fomit-frame-pointer
            -ffast-math
            -DNDEBUG
    )
endif()

# CUDA debug flags - simplified
set(CUDA_DEBUG_FLAGS
        -G
        -g
        -lineinfo
)

# CUDA release flags
set(CUDA_RELEASE_FLAGS
        -O3
        --use_fast_math
        -DNDEBUG
)

# Add models directory definition
add_definitions(-DLIGHTGLUE_MODELS_DIR="${LIGHTGLUE_MODELS_DIR}")

# Source files
set(LIGHTGLUE_HEADERS
        include/feature/ALIKED.hpp
        include/feature/DKD.hpp
        include/feature/SDDH.hpp
        include/feature/blocks.hpp
        include/feature/get_patches.hpp
        include/feature/input_padder.hpp
        include/feature/deform_conv2d.h
        include/feature/get_patches_cuda.h
        include/feature/cuda_helpers.h

        include/matcher/lightglue/attention.hpp
        include/matcher/lightglue/core.hpp
        include/matcher/lightglue/encoding.hpp
        include/matcher/lightglue/matcher.hpp
        include/matcher/lightglue/transformer.hpp
)

set(LIGHTGLUE_SOURCES
        src/feature/ALIKED.cpp
        src/feature/DKD.cpp
        src/feature/input_padder.cpp
        src/feature/get_patches.cpp
        src/feature/SDDH.cpp
        src/feature/deform_conv2d.cpp
        src/feature/deform_conv2d_kernel.cu
        src/feature/get_patches_cuda.cu

        src/matcher/lightglue/attention.cpp
        src/matcher/lightglue/core.cpp
        src/matcher/lightglue/encoding.cpp
        src/matcher/lightglue/matcher.cpp
        src/matcher/lightglue/transformer.cpp
        src/feature/blocks.cpp
)

# Library target
add_library(${PROJECT_NAME}_lib STATIC
        ${LIGHTGLUE_SOURCES}
        ${LIGHTGLUE_HEADERS}
)

add_library(${PROJECT_NAME}::lib ALIAS ${PROJECT_NAME}_lib)

target_include_directories(${PROJECT_NAME}_lib
        PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

# Configure compile options based on build type - simplified
target_compile_options(${PROJECT_NAME}_lib
        PRIVATE
        $<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CONFIG:Debug>>:${DEBUG_FLAGS}>
        $<$<AND:$<COMPILE_LANGUAGE:CXX>,$<CONFIG:Release>>:${RELEASE_FLAGS}>
        $<$<AND:$<COMPILE_LANGUAGE:CUDA>,$<CONFIG:Debug>>:${CUDA_DEBUG_FLAGS}>
        $<$<AND:$<COMPILE_LANGUAGE:CUDA>,$<CONFIG:Release>>:${CUDA_RELEASE_FLAGS}>
)

target_link_libraries(${PROJECT_NAME}_lib
        PUBLIC
        ${TORCH_LIBRARIES}
        ${OpenCV_LIBS}
        PRIVATE
        CUDA::cudart
        CUDA::curand
        CUDA::cublas
)

# Properties for debug/release configurations
set_target_properties(${PROJECT_NAME}_lib PROPERTIES
        CUDA_SEPARABLE_COMPILATION ON
        CUDA_RESOLVE_DEVICE_SYMBOLS ON
        POSITION_INDEPENDENT_CODE ON
        DEBUG_POSTFIX "d"
)

# Debug configuration specific settings - removed problematic definitions
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_definitions(${PROJECT_NAME}_lib
            PRIVATE
            DEBUG
    )
endif()

# Example application
add_executable(${PROJECT_NAME} examples/main.cpp)
target_link_libraries(${PROJECT_NAME} PRIVATE ${PROJECT_NAME}::lib)

# Output directories with debug/release suffixes
set_target_properties(${PROJECT_NAME} ${PROJECT_NAME}_lib PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY_DEBUG "${CMAKE_BINARY_DIR}/bin/debug"
        RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/bin/release"
        LIBRARY_OUTPUT_DIRECTORY_DEBUG "${CMAKE_BINARY_DIR}/lib/debug"
        LIBRARY_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/lib/release"
        ARCHIVE_OUTPUT_DIRECTORY_DEBUG "${CMAKE_BINARY_DIR}/lib/debug"
        ARCHIVE_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/lib/release"
)

# Create models directory if it doesn't exist
add_custom_target(create_models_dir ALL
        COMMAND ${CMAKE_COMMAND} -E make_directory ${LIGHTGLUE_MODELS_DIR}
)

# Print configuration information
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "Models directory: ${LIGHTGLUE_MODELS_DIR}")

# =========================
# Install / export for reuse
# =========================
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Option: build shared library (better for reuse) instead of static
option(LIGHTGLUE_BUILD_SHARED "Build LightGlue as a shared library" OFF)
if(LIGHTGLUE_BUILD_SHARED)
    set_target_properties(${PROJECT_NAME}_lib PROPERTIES
        POSITION_INDEPENDENT_CODE ON
    )
endif()

# Make sure consumers get required include dirs
target_include_directories(${PROJECT_NAME}_lib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# (Optional but recommended) expose C++ standard to consumers
target_compile_features(${PROJECT_NAME}_lib PUBLIC cxx_std_20)

# Export name for consumers
set_target_properties(${PROJECT_NAME}_lib PROPERTIES
    EXPORT_NAME lib
)

# Install the library + executable (exe optional)
install(TARGETS ${PROJECT_NAME}_lib
    EXPORT LightGlueTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Install public headers
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)

# Export targets file
install(EXPORT LightGlueTargets
    FILE LightGlueTargets.cmake
    NAMESPACE LightGlue::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/LightGlue
)

# Generate and install Config + Version files
set(LIGHTGLUE_CONFIG_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR}/cmake/LightGlue)

configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/LightGlueConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/LightGlueConfig.cmake
    INSTALL_DESTINATION ${LIGHTGLUE_CONFIG_INSTALL_DIR}
)

write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/LightGlueConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/LightGlueConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/LightGlueConfigVersion.cmake
    DESTINATION ${LIGHTGLUE_CONFIG_INSTALL_DIR}
)
