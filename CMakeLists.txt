cmake_minimum_required(VERSION 3.7)

project(libxml2_vendor)

find_package(ament_cmake REQUIRED)

# libxml2 is the GNOME XML library. ROS2 pulls it in via libxslt / Python lxml
# and a few XML consumers. lyrical targets Ubuntu 26.04 (resolute), whose
# libxml2-dev is 2.15.2 — pin the upstream GNOME tag to match.

option(FORCE_BUILD_VENDOR_PKG
  "Build libxml2 from source, even if a system-installed package is available"
  OFF)

if(NOT FORCE_BUILD_VENDOR_PKG)
  find_package(LibXml2 QUIET)
endif()

macro(build_libxml2)
  set(extra_cmake_args)
  if(DEFINED CMAKE_BUILD_TYPE)
    list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  endif()
  if(DEFINED BUILD_SHARED_LIBS)
    list(APPEND extra_cmake_args -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS})
  else()
    list(APPEND extra_cmake_args -DBUILD_SHARED_LIBS=ON)
  endif()
  if(DEFINED CMAKE_TOOLCHAIN_FILE)
    list(APPEND extra_cmake_args "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}")
  endif()
  list(APPEND extra_cmake_args "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}")
  # VxWorks: no iconv / liblzma / python in the target toolchain (matches pkg/libxml2).
  list(APPEND extra_cmake_args
    -DLIBXML2_WITH_ICONV=OFF
    -DLIBXML2_WITH_LZMA=OFF
    -DLIBXML2_WITH_PYTHON=OFF
    -DLIBXML2_WITH_ZLIB=OFF
    -DLIBXML2_WITH_TESTS=OFF
    -DLIBXML2_WITH_PROGRAMS=OFF)

  include(ExternalProject)

  # VxWorks source patch (carried from pkg/libxml2/files/0001-add-VxWorks-support.patch):
  #  - CMakeLists.txt: disable ICONV/LZMA defaults that don't build on VxWorks.
  externalproject_add(libxml2_external
    GIT_REPOSITORY https://github.com/GNOME/libxml2.git
    GIT_TAG v2.15.2
    GIT_CONFIG advice.detachedHead=false
    UPDATE_COMMAND ""
    TIMEOUT 600
    PATCH_COMMAND
      patch -p1 -N -r - < ${CMAKE_CURRENT_SOURCE_DIR}/patches/0001-add-VxWorks-support.patch || true
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install
      ${extra_cmake_args}
  )

  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}
    USE_SOURCE_PERMISSIONS
  )
endmacro()

if(NOT LibXml2_FOUND)
  message(STATUS "Did not find libxml2 installation, building from source")
  build_libxml2()
endif()

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
