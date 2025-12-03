# Copyright (c) 2023-2025, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

cmake_minimum_required (VERSION 3.31.8)

project(triton-bindings LANGUAGES C CXX)

# Top level module entry point and typed marker
file(COPY __init__.py DESTINATION .)
file(COPY py.typed DESTINATION .)
# Copy the '__init__.py' for the '_c' module
file(COPY _c/__init__.py DESTINATION ./_c/.)
# Find and copy _api modules
file(GLOB PYTHON_MODULE_FILES ./_api/*.py)
file(COPY ${PYTHON_MODULE_FILES} DESTINATION ./_api/.)

# include(FetchContent)
# FetchContent_Declare(
#   pybind11
#   GIT_REPOSITORY "https://github.com/pybind/pybind11"
#   # COMMIT ID for v2.12.0
#   GIT_TAG "3e9dfa2866941655c56877882565e7577de6fc7b"
#   GIT_SHALLOW ON
# )

# FetchContent_MakeAvailable(pybind11)
# ...existing code...
# 替换原有 FetchContent_Declare/FetchContent_MakeAvailable + 直接 apply 补丁 的逻辑
find_program(GIT_EXECUTABLE git)
if(NOT GIT_EXECUTABLE)
  message(FATAL_ERROR "git not found; required to fetch and patch pybind11")
endif()

set(PYBIND11_GIT "https://github.com/pybind/pybind11")
set(PYBIND11_COMMIT "3e9dfa2866941655c56877882565e7577de6fc7b") # 固定提交
set(PYBIND11_SRC_DIR "${CMAKE_BINARY_DIR}/_deps/pybind11-src")
set(PYBIND11_BUILD_DIR "${CMAKE_BINARY_DIR}/_deps/pybind11-build")
set(PYBIND11_PATCH "${CMAKE_SOURCE_DIR}/cmake/patches/pybind11-fix.patch")

# 如果源码不存在则克隆并检出指定提交（depth 可选）
if(NOT EXISTS "${PYBIND11_SRC_DIR}/.git")
  execute_process(
    COMMAND ${GIT_EXECUTABLE} clone ${PYBIND11_GIT} "${PYBIND11_SRC_DIR}"
    RESULT_VARIABLE _git_clone_res
    OUTPUT_VARIABLE _git_clone_out
    ERROR_VARIABLE _git_clone_err
  )
  if(NOT _git_clone_res EQUAL 0)
    message(FATAL_ERROR "Failed to clone pybind11: ${_git_clone_err}")
  endif()

  execute_process(
    COMMAND ${GIT_EXECUTABLE} -C "${PYBIND11_SRC_DIR}" checkout ${PYBIND11_COMMIT}
    RESULT_VARIABLE _git_checkout_res
    OUTPUT_VARIABLE _git_checkout_out
    ERROR_VARIABLE _git_checkout_err
  )
  if(NOT _git_checkout_res EQUAL 0)
    message(FATAL_ERROR "Failed to checkout pybind11 commit: ${_git_checkout_err}")
  endif()
endif()

# 应用补丁（git apply），工作目录为源码目录
if(EXISTS "${PYBIND11_PATCH}")
  execute_process(
    COMMAND ${GIT_EXECUTABLE} -C "${PYBIND11_SRC_DIR}" apply "${PYBIND11_PATCH}"
    RESULT_VARIABLE _git_apply_result
    OUTPUT_VARIABLE _git_apply_out
    ERROR_VARIABLE _git_apply_err
  )
  if(NOT _git_apply_result EQUAL 0)
    message(FATAL_ERROR "Failed to apply pybind11 patch: ${_git_apply_err}")
  endif()
else()
  message(WARNING "pybind11 patch not found: ${PYBIND11_PATCH}")
endif()

# 将本地修改后的源码以子目录方式加入构建（替代 FetchContent_MakeAvailable）
add_subdirectory(${PYBIND11_SRC_DIR} ${PYBIND11_BUILD_DIR})
# ...existing code...

set(
  PYTHON_BINDING_SRCS
  _c/tritonserver_pybind.cc
)

pybind11_add_module(python-bindings SHARED ${PYTHON_BINDING_SRCS})
target_link_libraries(
  python-bindings
  PRIVATE
  triton-core-serverapi           # from repo-core
  triton-core-serverstub          # from repo-core
)
target_compile_features(python-bindings PRIVATE cxx_std_17)

set_property(TARGET python-bindings PROPERTY OUTPUT_NAME triton_bindings)
# Add Triton library default path in 'rpath' for runtime library lookup
set_target_properties(python-bindings PROPERTIES BUILD_RPATH "$ORIGIN:/opt/tritonserver/lib")
set_target_properties(python-bindings PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/python/tritonserver/_c/)
