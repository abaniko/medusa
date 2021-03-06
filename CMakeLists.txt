cmake_minimum_required(VERSION 3.1)

# project name

project(Medusa)

# medusa version

set(VERSION_MAJOR 0)
set(VERSION_MINOR 4)
set(VERSION_PATCH 4)

# options

option(MEDUSA_BUILD_TOOLS
  "Build Medusa tools"
  TRUE)

option(MEDUSA_BUILD_TESTS
  "Build Medusa tests"
  TRUE)

option(MEDUSA_CMAKE_USE_COTIRE
  "use cotire"
  FALSE)

option(MEDUSA_BUILD_WITH_OGDF
  "OGDF library allows medusa to display graph"
  FALSE)

option(MEDUSA_BUILD_WITH_LLVM
  "LLVM library allows medusa to compile to code (emul_llvm)"
  FALSE)

# variables

set(MEDUSA_MODULES_TO_BUILD "all"
  CACHE STRING "Semicolon-separated list of modules to build, or \"all\"")
list(REMOVE_DUPLICATES MEDUSA_MODULES_TO_BUILD)
list(SORT MEDUSA_MODULES_TO_BUILD)

if ("${MEDUSA_MODULES_TO_BUILD}" MATCHES "all")
  message("Compile all modules")
else()
  foreach (medusa_module ${MEDUSA_MODULES_TO_BUILD})
    # TODO(wisk): check if the module if correct
    message("Compile module ${medusa_module}")
  endforeach()
endif()

set(MEDUSA_EXECUTABLE_PATH bin)
set(MEDUSA_LIBRARY_PATH lib)
set(MEDUSA_MODULE_PATH lib/medusa)
set(MEDUSA_RESOURCE_PATH share/medusa)

## cotire

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
if (MEDUSA_CMAKE_USE_COTIRE)
  include(cotire) # https://github.com/sakra/cotire
  message("Cotire was enabled")
endif()

if (MEDUSA_BUILD_TESTS)
  enable_testing()
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# features

include(ExternalProject)

## C++
set(CMAKE_CXX_STANDARD 11)
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG(-fabi-version=9 HAVE_ABI_VERSION)
if (HAVE_ABI_VERSION)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fabi-version=9")
endif()
## CHECK_CXX_COMPILER_FLAG(-fvisibility=hidden HAVE_VISIBILITY_HIDDEN)
## if (HAVE_VISIBILITY_HIDDEN)
##   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")
## endif()
## CHECK_CXX_COMPILER_FLAG(-fvisibility-inlines-hidden HAVE_VISIBILITY_INLINES_HIDDEN)
## if (HAVE_VISIBILITY_INLINES_HIDDEN)
##   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility-inlines-hidden")
## endif()

set(MEDUSA_NEEDED_LIBRARIES)

## thread

find_package(Threads REQUIRED)
list(APPEND MEDUSA_NEEDED_LIBRARIES ${CMAKE_THREAD_LIBS_INIT})

## Boost

set(MEDUSA_BOOST_MIN_VERSION 1.54)

set(Boost_USE_STATIC_LIBS TRUE)
find_package(Boost ${MEDUSA_BOOST_MIN_VERSION} COMPONENTS system filesystem thread date_time program_options unit_test_framework REQUIRED)

list(APPEND MEDUSA_NEEDED_LIBRARIES ${Boost_LIBRARIES})

## OGDF

if (MEDUSA_BUILD_WITH_OGDF)
  message("Build medusa with OGDF")
  include(ExternalProject)

  ExternalProject_add(ogdf
    PREFIX          "ogdf"

    GIT_REPOSITORY  "https://github.com/ogdf/ogdf"
    GIT_TAG         "master"

    SOURCE_DIR      "${CMAKE_SOURCE_DIR}/deps/ogdf"
    INSTALL_DIR     "${CMAKE_BINARY_DIR}"

    UPDATE_COMMAND  ""
    INSTALL_COMMAND ""
  )

  set(MEDUSA_HAS_OGDF "#define MEDUSA_HAS_OGDF")
  set(OGDF_INCLUDE_DIRS
    "${CMAKE_SOURCE_DIR}/deps/ogdf/include"
    "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/include")
  if (MSVC)
    set(OGDF_LIBRARIES
      debug     "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Debug/COIN.lib"
      debug     "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Debug/OGDF.lib"
      optimized "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Release/COIN.lib"
      optimized "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Release/OGDF.lib"
    )

    list(APPEND MEDUSA_NEEDED_LIBRARIES debug "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Debug/COIN.lib")
    list(APPEND MEDUSA_NEEDED_LIBRARIES debug "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Debug/OGDF.lib")
    list(APPEND MEDUSA_NEEDED_LIBRARIES optimized "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Release/COIN.lib")
    list(APPEND MEDUSA_NEEDED_LIBRARIES optimized "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/Release/OGDF.lib")
  else()
    set(OGDF_LIBRARIES
      "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/libCOIN.a"
      "${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/libOGDF.a"
    )
    list(APPEND MEDUSA_NEEDED_LIBRARIES ${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/libOGDF.a)
    list(APPEND MEDUSA_NEEDED_LIBRARIES ${CMAKE_BINARY_DIR}/ogdf/src/ogdf-build/libCOIN.a)
  endif()

endif()

## LLVM

if (MEDUSA_BUILD_WITH_LLVM)

  #set(LLVM_ROOT ${INSTALL_DIR})
  #set(LLVM_ROOT_DEBUG ${INSTALL_DIR})
  #set(LLVM_ROOT_RELEASE ${INSTALL_DIR})
endif()

# all generated files are going to bin folder
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# add include directory
include_directories(
  ${CMAKE_SOURCE_DIR}/inc
  ${CMAKE_BINARY_DIR}/inc
  ${Boost_INCLUDE_DIRS}
  )

if (MEDUSA_BUILD_WITH_OGDF)
  include_directories("${OGDF_INCLUDE_DIRS}")
endif()

link_directories(
  ${Boost_LIBRARY_DIRS}
  )

# src: ogdf cmake
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  add_definitions(
    "/DNOMINMAX"
    "/D_CRT_SECURE_NO_WARNINGS"
    "/D_SCL_SECURE_NO_WARNINGS"
    "/wd4251" "/wd4503"
    "/Zm256")
endif()

# add source directory
add_subdirectory(src)

# add packaging directory
add_subdirectory(package)

find_package(Doxygen)
if (DOXYGEN_FOUND)
  message("Doxygen found, I'll generate documentations for you")
  set(DOCROOT "${CMAKE_SOURCE_DIR}/doc/api")

  configure_file("${DOCROOT}/Doxyfile.in" "${DOCROOT}/Doxyfile" @ONLY)

  set(DOC_TARGET_NAME "doc")
  if (MSVC_IDE)
    set(DOC_TARGET_NAME "DOCUMENTATION")
  endif()

  add_custom_target(${DOC_TARGET_NAME}
    ${DOXYGEN_EXECUTABLE} ${DOCROOT}/Doxyfile
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Generating API documentation with doxygen" VERBATIM
    )
endif()

configure_file(
  ${CMAKE_SOURCE_DIR}/inc/medusa/version.hpp.in
  ${CMAKE_BINARY_DIR}/inc/medusa/version.hpp
)
