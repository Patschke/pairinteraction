# SPDX-FileCopyrightText: 2024 Pairinteraction Developers
# SPDX-License-Identifier: LGPL-3.0-or-later

# ------------------------------------------------------------
# Generate the backend target
# ------------------------------------------------------------

# nanobind_add_module backend (this internally calls cmake add_library(backend ...))
nanobind_add_module(
  backend
  STABLE_ABI
  NB_STATIC
  # cmake-format: off
  NB_DOMAIN pairinteraction
  # cmake-format: on
  basis/Basis.py.hpp
  basis/Basis.py.cpp
  database/Database.py.hpp
  database/Database.py.cpp
  diagonalize/Diagonalizer.py.cpp
  diagonalize/Diagonalizer.py.hpp
  enums/FloatType.py.hpp
  enums/FloatType.py.cpp
  enums/OperatorType.py.hpp
  enums/OperatorType.py.cpp
  enums/Parity.py.hpp
  enums/Parity.py.cpp
  enums/TransformationType.py.hpp
  enums/TransformationType.py.cpp
  interfaces/DiagonalizerInterface.py.hpp
  interfaces/DiagonalizerInterface.py.cpp
  interfaces/TransformationBuilderInterface.py.hpp
  interfaces/TransformationBuilderInterface.py.cpp
  ket/Ket.py.hpp
  ket/Ket.py.cpp
  operator/Operator.py.hpp
  operator/Operator.py.cpp
  system/System.py.hpp
  system/System.py.cpp
  tools/run_unit_tests.py.hpp
  tools/run_unit_tests.py.cpp
  bindings.cpp
  LoggerBridge.cpp
  LoggerBridge.hpp
  paths.py.hpp
  paths.py.cpp
  version.py.hpp
  version.py.cpp)

target_link_libraries(backend PRIVATE pairinteraction $<$<BOOL:${WITH_COVERAGE}>:coverage::gcov>)

set_target_properties(
  backend
  PROPERTIES OUTPUT_NAME _backend
             INSTALL_RPATH $<$<BOOL:${SKBUILD}>:$<$<PLATFORM_ID:Linux>:$ORIGIN>$<$<PLATFORM_ID:Darwin>:@loader_path>>)

# We mark nanobind's include directory as system include directory to suppress warnings and avoid clang-tidy checks
target_include_directories(backend SYSTEM PRIVATE ${NB_DIR}/include)

# Windows-specific post-build command to copy runtime DLLs
if(WIN32)
  add_custom_command(
    TARGET backend
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_RUNTIME_DLLS:backend> $<TARGET_FILE_DIR:backend>
    COMMAND_EXPAND_LISTS)
endif()

install(IMPORTED_RUNTIME_ARTIFACTS duckdb DESTINATION pairinteraction)
install(IMPORTED_RUNTIME_ARTIFACTS TBB::tbb DESTINATION pairinteraction)

# HACK: Under Apple, we cross-compile for arm64 on a x86_64 platform. The resulting python package cannot be imported
# and a stub file cannot be generated. For the published wheels we include the stub files from src/pairinteraction which
# are generated by the second install below in a previous build.
if(NOT DEFINED ENV{CIBW_ARCHS_MACOS} AND NOT CMAKE_CXX_FLAGS MATCHES "sanitize=address")
  # cmake-format: off
  nanobind_add_stub(
    backend_stub
    MODULE _backend
    OUTPUT _backend.pyi
    PYTHON_PATH $<TARGET_FILE_DIR:backend>
    DEPENDS backend
  )
  # cmake-format: on
  # Add mypy ignore comment at the start of the stub file
  install(CODE "file(READ ${CMAKE_CURRENT_BINARY_DIR}/_backend.pyi CONTENT)
          file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/_backend.pyi \"# mypy: ignore-errors\n\n\${CONTENT}\")")
  # Install the .pyi file to the python package directory
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/_backend.pyi DESTINATION pairinteraction)
  # Install the .pyi file also to the python src dir to make static type checking possible with pip editable installs
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/_backend.pyi DESTINATION ${CMAKE_SOURCE_DIR}/src/pairinteraction)
endif()

# ------------------------------------------------------------
# Generate the info target
# ------------------------------------------------------------

# nanobind_add_module info (this internally calls cmake add_library(info ...))
nanobind_add_module(
  info
  STABLE_ABI
  NB_STATIC
  # cmake-format: off
  NB_DOMAIN pairinteraction
  # cmake-format: on
  bindings_info.cpp
  Info.py.hpp
  Info.py.cpp)

target_include_directories(info PUBLIC $<TARGET_PROPERTY:pairinteraction,INTERFACE_INCLUDE_DIRECTORIES>)
target_compile_definitions(info PUBLIC $<TARGET_PROPERTY:pairinteraction,INTERFACE_COMPILE_DEFINITIONS>)

set_target_properties(
  info
  PROPERTIES OUTPUT_NAME _info)

# We mark nanobind's include directory as system include directory to suppress warnings and avoid clang-tidy checks
target_include_directories(info SYSTEM PRIVATE ${NB_DIR}/include)

# HACK: Under Apple, we cross-compile for arm64 on a x86_64 platform. The resulting python package cannot be imported
# and a stub file cannot be generated. For the published wheels we include the stub files from src/pairinteraction which
# are generated by the second install below in a previous build.
if(NOT DEFINED ENV{CIBW_ARCHS_MACOS} AND NOT CMAKE_CXX_FLAGS MATCHES "sanitize=address")
  # cmake-format: off
  nanobind_add_stub(
    info_stub
    MODULE _info
    OUTPUT _info.pyi
    PYTHON_PATH $<TARGET_FILE_DIR:info>
    DEPENDS info
  )
  # cmake-format: on
  # Add mypy ignore comment at the start of the stub file
  install(CODE "file(READ ${CMAKE_CURRENT_BINARY_DIR}/_info.pyi CONTENT)
          file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/_info.pyi \"# mypy: ignore-errors\n\n\${CONTENT}\")")
  # Install the .pyi file to the python package directory
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/_info.pyi DESTINATION pairinteraction)
  # Install the .pyi file also to the python src dir to make static type checking possible with pip editable installs
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/_info.pyi DESTINATION ${CMAKE_SOURCE_DIR}/src/pairinteraction)
endif()

# ------------------------------------------------------------
# Ensure that clang-tidy is really not run on nanobind
# ------------------------------------------------------------

# HACK: Under some circumstances, the marking nanobind's include directory as system include directory does not seem
# to avoid clang-tidy checks. We therefore disable clang-tidy manually for the nanobind-static target that is built
# within nanobind_add_module.
if((Python_VERSION VERSION_LESS 3.12)
  OR (NOT Python_INTERPRETER_ID STREQUAL "Python")
  OR (NOT TARGET Python::SABIModule))
 set_target_properties(nanobind-static PROPERTIES CXX_CLANG_TIDY "")
else()
 set_target_properties(nanobind-static-abi3 PROPERTIES CXX_CLANG_TIDY "")
endif()

# ------------------------------------------------------------
# Install the targets
# ------------------------------------------------------------

install(TARGETS backend info DESTINATION pairinteraction)
