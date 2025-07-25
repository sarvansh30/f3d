cmake_minimum_required(VERSION 3.21)

project(pyf3d)

option(F3D_BINDINGS_PYTHON_GENERATE_STUBS "Generate Python stubs" OFF)

if (WIN32)
  set(F3D_BINDINGS_PYTHON_WINDOWS_BUILD_DLL_FIXUP "" CACHE STRING "Path to folder containing DLLs to point to when using the Python wheels from build")
  mark_as_advanced(F3D_BINDINGS_PYTHON_WINDOWS_BUILD_DLL_FIXUP)
endif ()

list(INSERT CMAKE_MODULE_PATH 0 "${CMAKE_CURRENT_LIST_DIR}/cmake")
include(GNUInstallDirs)
include(f3dPython)

find_package(Python 3.9 COMPONENTS Interpreter Development)
find_package(pybind11 2.9.2 REQUIRED)

pybind11_add_module(pyf3d MODULE F3DPythonBindings.cxx)

# In case pyf3d is built separately, we need to retrieve the existing libf3d
if(PROJECT_IS_TOP_LEVEL)
  find_package(f3d REQUIRED COMPONENTS library)
  set(F3D_VERSION "${f3d_VERSION}")
  target_link_libraries(pyf3d PRIVATE f3d::libf3d)
  target_include_directories(pyf3d PRIVATE "${f3d_INCLUDE_DIR}/f3d")
else()
  target_link_libraries(pyf3d PRIVATE libf3d)
endif()

set(f3d_python_package_name "f3d")
set(f3d_python_package_suffix "python${Python_VERSION_MAJOR}.${Python_VERSION_MINOR}/site-packages")

get_property(F3D_MULTI_CONFIG_GENERATOR GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
set(f3d_binary_dir "${CMAKE_BINARY_DIR}$<${F3D_MULTI_CONFIG_GENERATOR}:/$<CONFIG>>")
set(f3d_module_dir "${f3d_binary_dir}/${f3d_python_package_name}")

set_target_properties(pyf3d PROPERTIES
  CXX_STANDARD 17
  CXX_VISIBILITY_PRESET hidden
  LIBRARY_OUTPUT_DIRECTORY "${f3d_module_dir}"
  )

if(SKBUILD)
  if(APPLE)
    set_target_properties(pyf3d PROPERTIES INSTALL_RPATH "@loader_path/${CMAKE_INSTALL_LIBDIR}")
  elseif(UNIX)
    set_target_properties(pyf3d PROPERTIES INSTALL_RPATH "$ORIGIN/${CMAKE_INSTALL_LIBDIR}")
  endif()
  set(f3d_python_install_path ".")
else()
  if(APPLE)
    set_target_properties(pyf3d PROPERTIES INSTALL_RPATH "@loader_path/../..")
  elseif(UNIX)
    set_target_properties(pyf3d PROPERTIES INSTALL_RPATH "$ORIGIN/../..")
  endif()
  set(f3d_python_install_path "${CMAKE_INSTALL_LIBDIR}/${f3d_python_package_suffix}/${f3d_python_package_name}")
endif()

# generate Windows fixup for build
set(F3D_ABSOLUTE_DLLS_FIXUP "")
set(F3D_RELATIVE_DLLS_FIXUP "")
if(WIN32)
  if(PROJECT_IS_TOP_LEVEL)
    f3d_python_windows_dll_fixup(PATHS "$<TARGET_FILE_DIR:f3d::libf3d>" OUTPUT F3D_ABSOLUTE_DLLS_FIXUP)
  else()
    set(f3d_win_dll_paths "${F3D_BINDINGS_PYTHON_WINDOWS_BUILD_DLL_FIXUP}" "$<TARGET_FILE_DIR:VTK::CommonCore>")
    if(BUILD_SHARED_LIBS)
      list(APPEND f3d_win_dll_paths "$<TARGET_FILE_DIR:libf3d>")
    endif()
    f3d_python_windows_dll_fixup(PATHS "${f3d_win_dll_paths}" OUTPUT F3D_ABSOLUTE_DLLS_FIXUP)
  endif()
endif()

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/__init__.py.in"
  "${CMAKE_CURRENT_BINARY_DIR}/__init__.py-build" @ONLY)

file(GENERATE
  OUTPUT "${f3d_module_dir}/__init__.py"
  INPUT "${CMAKE_CURRENT_BINARY_DIR}/__init__.py-build"
)

# generate Windows fixup for install
set(F3D_ABSOLUTE_DLLS_FIXUP "")
set(F3D_RELATIVE_DLLS_FIXUP "")
if(WIN32)
  if(SKBUILD)
    set(f3d_python_dll_install_bin_path "${CMAKE_INSTALL_BINDIR}")
  else()
    set(f3d_python_dll_install_bin_path "../../../${CMAKE_INSTALL_BINDIR}")
  endif()
  f3d_python_windows_dll_fixup(PATHS "${f3d_python_dll_install_bin_path}" OUTPUT F3D_RELATIVE_DLLS_FIXUP)
endif()

configure_file("${CMAKE_CURRENT_SOURCE_DIR}/__init__.py.in"
  "${CMAKE_CURRENT_BINARY_DIR}/__init__.py-install" @ONLY)

# Stubs
if (F3D_BINDINGS_PYTHON_GENERATE_STUBS)
  add_custom_command(
    TARGET pyf3d
    POST_BUILD
    COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/generate_stubs.py --into=${f3d_binary_dir}
    WORKING_DIRECTORY ${f3d_binary_dir})
endif()

# testing
if(BUILD_TESTING)
  add_subdirectory(testing)
endif()

# installing
install(TARGETS pyf3d
  LIBRARY DESTINATION ${f3d_python_install_path} COMPONENT python)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/__init__.py-install" RENAME "__init__.py"
  DESTINATION ${f3d_python_install_path} COMPONENT python)
if (F3D_BINDINGS_PYTHON_GENERATE_STUBS)
  install(FILES "${f3d_module_dir}/__init__.pyi" "${f3d_module_dir}/pyf3d.pyi" "${CMAKE_CURRENT_SOURCE_DIR}/py.typed"
    DESTINATION ${f3d_python_install_path} COMPONENT python)
endif()
