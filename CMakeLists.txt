# Copyright 2021-2022, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

cmake_minimum_required(VERSION 3.17)

project(tritonopenvinobackend LANGUAGES C CXX)

#
# Set TRITON_BUILD_OPENVINO_VERSION to the version of OpenVINO that
# you want to be built for the backend.
#
# Set TRITON_BUILD_CONTAINER to the Triton container to use as a base
# for the build. Required on Windows. Required on Linux unless
# TRITON_BUILD_CONTAINER_VERSION is set.
#
# On linux, in place of TRITON_BUILD_CONTAINER, you can set
# TRITON_BUILD_CONTAINER_VERSION to the Triton version that you want
# to target with the build and the corresponding container from NGC
# will be used.
#
option(TRITON_ENABLE_STATS "Include statistics collections in backend" ON)
set(TRITON_BUILD_CONTAINER "" CACHE STRING "Triton container to use a base for build")
set(TRITON_BUILD_CONTAINER_VERSION "" CACHE STRING "Triton container version to target")
set(TRITON_BUILD_OPENVINO_VERSION "" CACHE STRING "OpenVINO version to build")
set(TRITON_BUILD_OPENVINO_COMMIT_SHA "" CACHE STRING "OpenVINO commit version to build")
set(TRITON_OPENVINO_BACKEND_INSTALLDIR "" CACHE STRING "The directory name to install openVINO backend")
set(TRITON_BUILD_USE_PREBUILT_OPENVINO OFF CACHE BOOL "Uses prebuilt OpenVINO libraries from Intel")

set(TRITON_BACKEND_REPO_TAG "main" CACHE STRING "Tag for triton-inference-server/backend repo")
set(TRITON_CORE_REPO_TAG "main" CACHE STRING "Tag for triton-inference-server/core repo")
set(TRITON_COMMON_REPO_TAG "main" CACHE STRING "Tag for triton-inference-server/common repo")

set(TRITON_ENABLE_GPU OFF CACHE BOOL "GPU disabled" FORCE)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()
set(CMAKE_BUILD_TYPE Release)

if(NOT TRITON_BUILD_OPENVINO_VERSION AND NOT TRITON_BUILD_OPENVINO_COMMIT_SHA)
  message(FATAL_ERROR "TRITON_BUILD_OPENVINO_VERSION or TRITON_BUILD_OPENVINO_COMMIT_SHA are required")
endif()

if(TRITON_BUILD_OPENVINO_COMMIT_SHA AND TRITON_BUILD_USE_PREBUILT_OPENVINO)
  message(FATAL_ERROR "TRITON_BUILD_USE_PREBUILT_OPENVINO can not be specified alongwith TRITON_BUILD_OPENVINO_COMMIT_SHA")
endif()

if(TRITON_BUILD_OPENVINO_COMMIT_SHA)
  set(TRITON_OPENVINO_BACKEND_LIBNAME triton_openvino_${TRITON_BUILD_OPENVINO_COMMIT_SHA})
  if(NOT TRITON_OPENVINO_BACKEND_INSTALLDIR) 
    set(TRITON_OPENVINO_BACKEND_INSTALLDIR openvino_${TRITON_BUILD_OPENVINO_COMMIT_SHA})
    message(STATUS " Set ${TRITON_OPENVINO_BACKEND_INSTALLDIR}")
  endif()
endif()

if(TRITON_BUILD_OPENVINO_VERSION MATCHES "^([0-9]+)[\.]([0-9]+)[\.0-9]*$")
  if(TRITON_BUILD_USE_PREBUILT_OPENVINO)
    set(TRITON_OPENVINO_BACKEND_LIBNAME triton_openvino_${CMAKE_MATCH_1}_${CMAKE_MATCH_2}_pre)
    if(NOT TRITON_OPENVINO_BACKEND_INSTALLDIR) 
      set(TRITON_OPENVINO_BACKEND_INSTALLDIR openvino_${CMAKE_MATCH_1}_${CMAKE_MATCH_2}_pre)
    endif()
  else()
    set(TRITON_OPENVINO_BACKEND_LIBNAME triton_openvino_${CMAKE_MATCH_1}_${CMAKE_MATCH_2})
    if(NOT TRITON_OPENVINO_BACKEND_INSTALLDIR) 
      set(TRITON_OPENVINO_BACKEND_INSTALLDIR openvino_${CMAKE_MATCH_1}_${CMAKE_MATCH_2})
    endif()
  endif()
else()
  if(TRITON_BUILD_OPENVINO_VERSION)
    message(FATAL_ERROR "TRITON_BUILD_OPENVINO_VERSION should be in the format <major>.<minor>[.<build_num>]")
  endif()
endif()

if(WIN32 AND NOT TRITON_BUILD_CONTAINER)
  message(FATAL_ERROR "TRITON_BUILD_CONTAINER is required")
endif()

if(WIN32 AND TRITON_BUILD_USE_PREBUILT_OPENVINO)
  message(FATAL_ERROR "TRITON_BUILD_USE_PREBUILT_OPENVINO not supported on WIN32")
endif()

if(NOT TRITON_BUILD_CONTAINER AND NOT TRITON_BUILD_CONTAINER_VERSION)
  message(FATAL_ERROR
    "TRITON_BUILD_CONTAINER or TRITON_BUILD_CONTAINER_VERSION is required")
endif()

if(NOT TRITON_BUILD_CONTAINER)
  set(TRITON_BUILD_CONTAINER "nvcr.io/nvidia/tritonserver:${TRITON_BUILD_CONTAINER_VERSION}-py3-min")
endif()

set(TRITON_OPENVINO_INCLUDE_PATHS "${CMAKE_CURRENT_BINARY_DIR}/openvino/include")
set(TRITON_OPENVINO_LIB_PATHS "${CMAKE_CURRENT_BINARY_DIR}/openvino/lib")
set(TRITON_OPENVINO_DOCKER_IMAGE "tritonserver_openvino")
set(TRITON_OPENVINO_DOCKER_MEMORY "16g")
if(TRITON_BUILD_OPENVINO_COMMIT_SHA)
  set(OPENVINO_LIBRARY_STUB "openvino")
else()
  set(OPENVINO_LIBRARY_STUB "inference_engine")
endif()
if (WIN32)
  if (${CMAKE_BUILD_TYPE} STREQUAL "Debug")
    set(OPENVINO_LIBRARY "${OPENVINO_LIBRARY_STUB}d.lib")
  else()
    set(OPENVINO_LIBRARY "${OPENVINO_LIBRARY_STUB}.lib")
  endif()
else()
  set(OPENVINO_LIBRARY "${OPENVINO_LIBRARY_STUB}.so")
endif() # WIN32

#
# Dependencies
#
# FetchContent's composibility isn't very good. We must include the
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
FetchContent_MakeAvailable(repo-common repo-core repo-backend)

#
# Shared library implementing the Triton Backend API
#
configure_file(src/libtriton_openvino.ldscript libtriton_openvino.ldscript COPYONLY)

add_library(
  triton-openvino-backend SHARED
  src/openvino.cc
  src/openvino_utils.cc
  src/openvino_utils.h
)

add_library(
  TritonOpenVINOBackend::triton-openvino-backend ALIAS triton-openvino-backend
)

target_include_directories(
  triton-openvino-backend
  PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${TRITON_OPENVINO_INCLUDE_PATHS}
)

target_compile_features(triton-openvino-backend PRIVATE cxx_std_11)
target_compile_options(
  triton-openvino-backend PRIVATE
  $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>:
    -Wall -Wextra -Wno-unused-parameter -Wno-type-limits -Wno-ignored-qualifiers -Werror>
  $<$<CXX_COMPILER_ID:MSVC>:/Wall /D_WIN32_WINNT=0x0A00 /EHsc>
)

set_target_properties(
  triton-openvino-backend
  PROPERTIES
    POSITION_INDEPENDENT_CODE ON
    OUTPUT_NAME ${TRITON_OPENVINO_BACKEND_LIBNAME}
    SKIP_BUILD_RPATH TRUE
    BUILD_WITH_INSTALL_RPATH TRUE
    INSTALL_RPATH_USE_LINK_PATH FALSE
    INSTALL_RPATH "$\{ORIGIN\}"
)

if(NOT WIN32)
  set_target_properties(
    triton-openvino-backend
    PROPERTIES
      LINK_DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/libtriton_openvino.ldscript
      LINK_FLAGS "-Wl,--version-script libtriton_openvino.ldscript"
  )
endif() # NOT WIN32

FOREACH(p ${TRITON_OPENVINO_LIB_PATHS})
  target_link_directories(
    triton-openvino-backend
    PRIVATE ${p}
  )
ENDFOREACH(p)

target_link_libraries(
  triton-openvino-backend
  PRIVATE
    triton-core-serverapi   # from repo-core
    triton-core-backendapi  # from repo-core
    triton-core-serverstub  # from repo-core
    triton-backend-utils    # from repo-backend
    ${OPENVINO_LIBRARY}
)

#
# Build the OpenVINO libraries using docker.
#
if(TRITON_BUILD_OPENVINO_COMMIT_SHA)
  set(OPENVINO_CPU_PLUGIN_NAME "libopenvino_intel_cpu_plugin.so")
else()
  set(OPENVINO_CPU_PLUGIN_NAME "libMKLDNNPlugin.so")
endif()
if (WIN32)
  add_custom_command(
    OUTPUT
      openvino/lib/${OPENVINO_LIBRARY}
    COMMAND python3 ${CMAKE_CURRENT_SOURCE_DIR}/tools/gen_openvino_dockerfile.py --build-type="${CMAKE_BUILD_TYPE}" --triton-container="${TRITON_BUILD_CONTAINER}" --openvino-version="${TRITON_BUILD_OPENVINO_VERSION}"  --openvino-commit-version="${TRITON_BUILD_OPENVINO_COMMIT_SHA}" --output=Dockerfile.openvino
    COMMAND docker build --memory ${TRITON_OPENVINO_DOCKER_MEMORY} --cache-from=${TRITON_OPENVINO_DOCKER_IMAGE} --cache-from=${TRITON_OPENVINO_DOCKER_IMAGE}_cache0 --cache-from=${TRITON_OPENVINO_DOCKER_IMAGE}_cache1 -t ${TRITON_OPENVINO_DOCKER_IMAGE} -f ./Dockerfile.openvino ${CMAKE_CURRENT_SOURCE_DIR}
    COMMAND powershell.exe -noprofile -c "docker rm openvino_backend_ov > $null 2>&1; if ($LASTEXITCODE) { 'error ignored...' }; exit 0"
    COMMAND docker create --name openvino_backend_ov ${TRITON_OPENVINO_DOCKER_IMAGE}
    COMMAND rmdir /s/q openvino
    COMMAND docker cp openvino_backend_ov:/opt/openvino openvino
    COMMAND docker rm openvino_backend_ov
    COMMENT "Building OpenVino"
  )
else()
  add_custom_command(
    OUTPUT
      openvino/lib/${OPENVINO_LIBRARY}
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/tools/gen_openvino_dockerfile.py --build-type="${CMAKE_BUILD_TYPE}" --triton-container="${TRITON_BUILD_CONTAINER}" --openvino-version="${TRITON_BUILD_OPENVINO_VERSION}" --use-prebuilt-openvino="${TRITON_BUILD_USE_PREBUILT_OPENVINO}" --openvino-commit-version="${TRITON_BUILD_OPENVINO_COMMIT_SHA}" --output=Dockerfile.openvino
    COMMAND docker build --cache-from=${TRITON_OPENVINO_DOCKER_IMAGE} --cache-from=${TRITON_OPENVINO_DOCKER_IMAGE}_cache0 --cache-from=${TRITON_OPENVINO_DOCKER_IMAGE}_cache1 -t ${TRITON_OPENVINO_DOCKER_IMAGE} -f ./Dockerfile.openvino ${CMAKE_CURRENT_SOURCE_DIR}
    COMMAND docker rm openvino_backend_ov || echo "error ignored..." || true
    COMMAND docker create --name openvino_backend_ov ${TRITON_OPENVINO_DOCKER_IMAGE}
    COMMAND rm -fr openvino
    COMMAND docker cp openvino_backend_ov:/opt/openvino openvino
    COMMAND docker rm openvino_backend_ov
    COMMAND echo '<ie><plugins><plugin name=\"CPU\" location=\"${OPENVINO_CPU_PLUGIN_NAME}\"></plugin></plugins></ie>' >> openvino/lib/plugins.xml
    COMMENT "Building OpenVino"
  )
endif() # WIN32

add_custom_target(ov_target DEPENDS openvino/lib/${OPENVINO_LIBRARY})
add_library(openvino-library SHARED IMPORTED GLOBAL)
add_dependencies(openvino-library ov_target)
add_dependencies(triton-openvino-backend openvino-library)
set_target_properties(
  openvino-library
  PROPERTIES
    IMPORTED_LOCATION openvino/lib/${OPENVINO_LIBRARY}
)

#
# Install
#
include(GNUInstallDirs)
set(INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/TritonOpenVINOBackend)

install(
  DIRECTORY
    ${CMAKE_CURRENT_BINARY_DIR}/openvino/
  DESTINATION ${CMAKE_INSTALL_PREFIX}/backends/${TRITON_OPENVINO_BACKEND_INSTALLDIR}
  PATTERN *lib EXCLUDE
  PATTERN *bin EXCLUDE
  PATTERN *include EXCLUDE
)

if (NOT WIN32)
  install(
    TARGETS
      triton-openvino-backend
    EXPORT
      triton-openvino-backend-targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/backends/${TRITON_OPENVINO_BACKEND_INSTALLDIR}
  )

  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/openvino/lib/
    USE_SOURCE_PERMISSIONS
    DESTINATION ${CMAKE_INSTALL_PREFIX}/backends/${TRITON_OPENVINO_BACKEND_INSTALLDIR}
  )
else()
  install(
    TARGETS
      triton-openvino-backend
    EXPORT
      triton-openvino-backend-targets
    RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/backends/${TRITON_OPENVINO_BACKEND_INSTALLDIR}
  )

  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/openvino/bin/
    USE_SOURCE_PERMISSIONS
    DESTINATION ${CMAKE_INSTALL_PREFIX}/backends/${TRITON_OPENVINO_BACKEND_INSTALLDIR}
  )
endif() # WIN32

install(
  EXPORT
    triton-openvino-backend-targets
  FILE
    TritonOpenVINOBackendTargets.cmake
  NAMESPACE
    TritonOpenVINOBackend::
  DESTINATION
    ${INSTALL_CONFIGDIR}
)

include(CMakePackageConfigHelpers)
configure_package_config_file(
  ${CMAKE_CURRENT_LIST_DIR}/cmake/TritonOpenVINOBackendConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/TritonOpenVINOBackendConfig.cmake
  INSTALL_DESTINATION ${INSTALL_CONFIGDIR}
)

install(
  FILES
  ${CMAKE_CURRENT_BINARY_DIR}/TritonOpenVINOBackendConfig.cmake
  DESTINATION ${INSTALL_CONFIGDIR}
)

#
# Export from build tree
#
export(
  EXPORT triton-openvino-backend-targets
  FILE ${CMAKE_CURRENT_BINARY_DIR}/TritonOpenVINOBackendTargets.cmake
  NAMESPACE TritonOpenVINOBackend::
)

export(PACKAGE TritonOpenVINOBackend)
