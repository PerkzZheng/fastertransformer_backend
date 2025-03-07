# Copyright (c) 2021-2022, NVIDIA CORPORATION. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required (VERSION 3.18)

project(tritonfastertransformerbackend LANGUAGES C CXX)

#
# Options
#
option(TRITON_ENABLE_GPU "Enable GPU support in backend" ON)
option(TRITON_ENABLE_STATS "Include statistics collections in backend" ON)
option(BUILD_MULTI_GPU "Enable multi GPU support" ON)

set(TRITON_PYTORCH_INCLUDE_PATHS "" CACHE PATH "Paths to Torch includes")
set(TRITON_PYTORCH_LIB_PATHS "" CACHE PATH "Paths to Torch libraries")

set(TRITON_BACKEND_REPO_TAG "r22.03" CACHE STRING "Tag for triton-inference-server/backend repo")
set(TRITON_CORE_REPO_TAG "r22.03" CACHE STRING "Tag for triton-inference-server/core repo")
set(TRITON_COMMON_REPO_TAG "r22.03" CACHE STRING "Tag for triton-inference-server/common repo")

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

set(USE_TRITONSERVER_DATATYPE "ON")
message("-- Enable USE_TRITONSERVER_DATATYPE")

set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/Modules)

# Python.h needed by torch headers.
find_package(Python3 REQUIRED COMPONENTS Development)

find_package(CUDA 10.1 REQUIRED)
if (BUILD_MULTI_GPU)
  message(STATUS "Enable BUILD_MULTI_GPU.")
  find_package(MPI REQUIRED)
  find_package(NCCL REQUIRED)
  message(STATUS "Found MPI (include: ${MPI_INCLUDE_DIRS}, library: ${MPI_LIBRARIES})")
  message(STATUS "Found NCCL (include: ${NCCL_INCLUDE_DIRS}, library: ${NCCL_LIBRARIES})")
else()
  message(STATUS "Disable BUILD_MULTI_GPU.")
endif()

if (${CUDA_VERSION} GREATER_EQUAL 11.0)
  message(STATUS "Add DCUDA11_MODE")
  add_definitions("-DCUDA11_MODE")
endif()

if(${CUDA_VERSION_MAJOR} VERSION_GREATER_EQUAL "11")
  add_definitions("-DENABLE_BF16")
  message("CUDA_VERSION ${CUDA_VERSION_MAJOR} is greater or equal than 11, enable -DENABLE_BF16 flag")
endif()

#
# Dependencies
#
# FetchContent's composability isn't very good. We must include the
# transitive closure of all repos so that we can override the tag.
#
include(FetchContent)

FetchContent_Declare(
  repo-common
  GIT_REPOSITORY https://github.com/triton-inference-server/common.git
  GIT_TAG ${TRITON_COMMON_REPO_TAG}
  GIT_SHALLOW ON
)
FetchContent_Declare(
  repo-core
  GIT_REPOSITORY https://github.com/triton-inference-server/core.git
  GIT_TAG ${TRITON_CORE_REPO_TAG}
  GIT_SHALLOW ON
)
FetchContent_Declare(
  repo-backend
  GIT_REPOSITORY https://github.com/triton-inference-server/backend.git
  GIT_TAG ${TRITON_BACKEND_REPO_TAG}
  GIT_SHALLOW ON
)
if (EXISTS ${FT_DIR})
  FetchContent_Declare(
    repo-ft
    SOURCE_DIR ${FT_DIR}
  )
else()
  FetchContent_Declare(
    repo-ft
    GIT_REPOSITORY https://github.com/NVIDIA/FasterTransformer.git
    GIT_TAG main
    GIT_SHALLOW ON
  )
endif()
FetchContent_MakeAvailable(repo-common repo-core repo-backend repo-ft)

#
# CUDA
#
if(${TRITON_ENABLE_GPU})
  find_package(CUDAToolkit REQUIRED)
endif() # TRITON_ENABLE_GPU

#
# Shared library implementing the Triton Backend API
#
configure_file(src/libtriton_fastertransformer.ldscript libtriton_fastertransformer.ldscript COPYONLY)

add_library(
  triton-fastertransformer-backend SHARED
  src/libfastertransformer.cc
)

add_library(
  TritonFasterTransformerBackend::triton-fastertransformer-backend ALIAS triton-fastertransformer-backend
)

#find_package(CUDAToolkit REQUIRED)
find_package(CUDA 10.1 REQUIRED)
if (${CUDA_VERSION} GREATER_EQUAL 11.0)
  message(STATUS "Add DCUDA11_MODE")
  add_definitions("-DCUDA11_MODE")
endif()

set(CUDA_PATH ${CUDA_TOOLKIT_ROOT_DIR})

target_compile_definitions(triton-fastertransformer-backend
  PUBLIC
  USE_TRITONSERVER_DATATYPE
  BUILD_MULTI_GPU)

target_include_directories(
  triton-fastertransformer-backend
  PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/src
  ${TRITON_PYTORCH_INCLUDE_PATHS}
  ${Python3_INCLUDE_DIRS}
  ${repo-ft_SOURCE_DIR}
  ${repo-ft_SOURCE_DIR}/3rdparty/cutlass/include
  ${repo-core_SOURCE_DIR}/include
  )

target_link_directories(
  triton-fastertransformer-backend
  PRIVATE
  ${CUDA_PATH}/lib64
  )

target_compile_features(triton-fastertransformer-backend PRIVATE cxx_std_14)
target_compile_options(
  triton-fastertransformer-backend PRIVATE
  $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>:
    -Wall -Wextra -Wno-unused-parameter -Wno-type-limits >#-Werror>
)

if(${TRITON_ENABLE_GPU})
  target_compile_definitions(
    triton-fastertransformer-backend
    PRIVATE TRITON_ENABLE_GPU=1
  )
endif() # TRITON_ENABLE_GPU

set_target_properties(
  triton-fastertransformer-backend
  PROPERTIES
    POSITION_INDEPENDENT_CODE ON
    OUTPUT_NAME triton_fastertransformer
    SKIP_BUILD_RPATH TRUE
    BUILD_WITH_INSTALL_RPATH TRUE
    INSTALL_RPATH_USE_LINK_PATH FALSE
    INSTALL_RPATH "$\{ORIGIN\}"
    LINK_DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/libtriton_fastertransformer.ldscript
    LINK_FLAGS "-Wl,--no-as-needed,--version-script libtriton_fastertransformer.ldscript"
)

# Need to turn off unused-but-set-variable due to Torchvision
# Need to turn off unknown-pragmas due to ATen OpenMP
set_target_properties(
  triton-fastertransformer-backend
  PROPERTIES COMPILE_FLAGS
    "-Wno-unknown-pragmas -Wno-unused-but-set-variable"
)

set(TRITON_PYTORCH_LDFLAGS "")
FOREACH(p ${TRITON_PYTORCH_LIB_PATHS})
  set(TRITON_PYTORCH_LDFLAGS ${TRITON_PYTORCH_LDFLAGS} "-L${p}")
ENDFOREACH(p)

target_link_libraries(
  triton-fastertransformer-backend
  PRIVATE
    triton-core-serverapi  # from repo-core
    triton-core-backendapi # from repo-core
    triton-core-serverstub # from repo-core
    triton-backend-utils   # from repo-backend
    transformer-shared     # from repo-ft
    ${TRITON_PYTORCH_LDFLAGS}
    -lcublas
    -lcublasLt
    -lcudart
    -lcurand
)

if (BUILD_MULTI_GPU)
  target_compile_definitions(
    triton-fastertransformer-backend
    PUBLIC
      BUILD_MULTI_GPU
  )
  target_include_directories(
    triton-fastertransformer-backend
    PRIVATE
      ${MPI_INCLUDE_PATH}
  )
  target_link_directories(
    triton-fastertransformer-backend
    PRIVATE
      ${MPI_Libraries}
      /usr/local/mpi/lib
  )
  target_link_libraries(
    triton-fastertransformer-backend
    PRIVATE
      ${NCCL_LIBRARIES}
      ${MPI_LIBRARIES}
  )
endif()

if(${TRITON_ENABLE_GPU})
  target_link_libraries(
    triton-fastertransformer-backend
    PRIVATE
      CUDA::cudart
  )
endif() # TRITON_ENABLE_GPU

#
# Install
#
include(GNUInstallDirs)
set(INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/TritonFasterTransformerBackend)

install(
  TARGETS
    triton-fastertransformer-backend
  EXPORT
    triton-fastertransformer-backend-targets
  LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/backends/fastertransformer
  ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/backends/fastertransformer
)

install(
  EXPORT
    triton-fastertransformer-backend-targets
  FILE
    TritonFasterTransformerBackendTargets.cmake
  NAMESPACE
    TritonFasterTransformerBackend::
  DESTINATION
    ${INSTALL_CONFIGDIR}
)

include(CMakePackageConfigHelpers)
configure_package_config_file(
  ${CMAKE_CURRENT_LIST_DIR}/cmake/TritonFasterTransformerBackendConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/TritonFasterTransformerBackendConfig.cmake
  INSTALL_DESTINATION ${INSTALL_CONFIGDIR}
)

install(
  FILES
  ${CMAKE_CURRENT_BINARY_DIR}/TritonFasterTransformerBackendConfig.cmake
  DESTINATION ${INSTALL_CONFIGDIR}
)

#
# Export from build tree
#
export(
  EXPORT triton-fastertransformer-backend-targets
  FILE ${CMAKE_CURRENT_BINARY_DIR}/TritonFasterTransformerBackendTargets.cmake
  NAMESPACE TritonFasterTransformerBackend::
)

export(PACKAGE TritonFasterTransformerBackend)
