CMAKE_MINIMUM_REQUIRED(VERSION 2.6)
PROJECT( octomap )

ENABLE_TESTING()

# version (e.g. for packaging)
set(OCTOMAP_MAJOR_VERSION 1)
set(OCTOMAP_MINOR_VERSION 8)
set(OCTOMAP_PATCH_VERSION 0)
set(OCTOMAP_VERSION ${OCTOMAP_MAJOR_VERSION}.${OCTOMAP_MINOR_VERSION}.${OCTOMAP_PATCH_VERSION})
set(OCTOMAP_SOVERSION ${OCTOMAP_MAJOR_VERSION}.${OCTOMAP_MINOR_VERSION})
if(COMMAND cmake_policy)
  cmake_policy(SET CMP0003 NEW)
endif(COMMAND cmake_policy)

SET (CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/CMakeModules")

# COMPILER SETTINGS (default: Release) and flags
INCLUDE(CompilerSettings)

# OCTOMAP_OMP = enable OpenMP parallelization (experimental, defaults to OFF)
SET(OCTOMAP_OMP FALSE CACHE BOOL "Enable/disable OpenMP parallelization")
IF(DEFINED ENV{OCTOMAP_OMP})
  SET(OCTOMAP_OMP $ENV{OCTOMAP_OMP})
ENDIF(DEFINED ENV{OCTOMAP_OMP})
IF(OCTOMAP_OMP)
  FIND_PACKAGE( OpenMP REQUIRED)
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
ENDIF(OCTOMAP_OMP)

# Set output directories for libraries and executables
SET( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib/octomap )
SET( CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib/octomap )
SET( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin/octomap )
# output dirs for multi-config builds (MSVC)
foreach( OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES} )
  STRING( TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG )
  SET( CMAKE_LIBRARY_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_BINARY_DIR}/lib/octomap )
  SET( CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_BINARY_DIR}/lib/octomap )
  SET( CMAKE_RUNTIME_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_BINARY_DIR}/bin/octomap )
endforeach( OUTPUTCONFIG CMAKE_CONFIGURATION_TYPES )

set(INCLUDE_DIRS "${PROJECT_SOURCE_DIR}/include")
INCLUDE_DIRECTORIES(${INCLUDE_DIRS})

LINK_DIRECTORIES(${CMAKE_LIBRARY_OUTPUT_DIRECTORY})

# Installation

set(INSTALL_TARGETS_DEFAULT_ARGS
	RUNTIME DESTINATION bin
	LIBRARY DESTINATION lib
	ARCHIVE DESTINATION lib
)

ADD_SUBDIRECTORY( src/math )
ADD_SUBDIRECTORY( src )

file(GLOB octomap_BINS ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/*)
install(FILES ${octomap_BINS} DESTINATION bin)
file(GLOB octomap_LIBS ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/*)
install(FILES ${octomap_LIBS} DESTINATION lib)
file(GLOB octomap_HDRS ${PROJECT_SOURCE_DIR}/include/octomap/*.h ${PROJECT_SOURCE_DIR}/include/octomap/*.hxx)
install(FILES ${octomap_HDRS}	DESTINATION include/octomap)
file(GLOB octomap_math_HDRS ${PROJECT_SOURCE_DIR}/include/octomap/math/*.h)
install(FILES ${octomap_math_HDRS}	DESTINATION include/octomap/math)

# Install catkin package.xml
install(FILES package.xml DESTINATION share/octomap)

# uninstall target
configure_file(
    "${PROJECT_SOURCE_DIR}/CMakeModules/CMakeUninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)

add_custom_target(uninstall
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
export(PACKAGE octomap)

# Create a octomap-config.cmake file for the use from the build tree
set(OCTOMAP_INCLUDE_DIRS "${INCLUDE_DIRS}")
set(OCTOMAP_LIB_DIR "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
# Set library names as absolute paths
# Windows, spec. MSVC requires the .lib suffix for imported libs
IF(WIN32)
  set(OCTOMAP_LIBRARY
    "${CMAKE_IMPORT_LIBRARY_PREFIX}octomap${CMAKE_IMPORT_LIBRARY_SUFFIX}"
  )
  set(OCTOMATH_LIBRARY
    "${CMAKE_IMPORT_LIBRARY_PREFIX}octomath${CMAKE_IMPORT_LIBRARY_SUFFIX}"
  )
ELSE()
  set(OCTOMAP_LIBRARY
    "${CMAKE_SHARED_LIBRARY_PREFIX}octomap${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )
  set(OCTOMATH_LIBRARY
    "${CMAKE_SHARED_LIBRARY_PREFIX}octomath${CMAKE_SHARED_LIBRARY_SUFFIX}"
  )
ENDIF()

set(OCTOMAP_INCLUDE_DIRS "${CMAKE_INSTALL_PREFIX}/include")
set(OCTOMAP_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib")

configure_file(octomap-config.cmake.in
  "${PROJECT_BINARY_DIR}/InstallFiles/octomap-config.cmake" @ONLY)
configure_file(octomap-config-version.cmake.in
  "${PROJECT_BINARY_DIR}/InstallFiles/octomap-config-version.cmake" @ONLY)
install(FILES
  "${PROJECT_BINARY_DIR}/InstallFiles/octomap-config.cmake"
  "${PROJECT_BINARY_DIR}/InstallFiles/octomap-config-version.cmake"
  DESTINATION share/octomap/)

# Write pkgconfig-file:
include(InstallPkgConfigFile)
install_pkg_config_file(octomap
    CFLAGS
    LIBS -loctomap -loctomath
    REQUIRES
    VERSION ${OCTOMAP_VERSION})

# Documentation
FIND_PACKAGE(Doxygen)
IF(DOXYGEN_FOUND)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/octomap.dox.in ${CMAKE_CURRENT_BINARY_DIR}/octomap.dox @ONLY)
  ADD_CUSTOM_TARGET(docs ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/octomap.dox
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    COMMENT "Generating documentation (Doxygen) at location ${PROJECT_SOURCE_DIR}/doc/html/")
ENDIF(DOXYGEN_FOUND)

# Needs to be last statement:
INCLUDE(CPackSettings)

# Finished:
MESSAGE ("\n")
MESSAGE (STATUS "Compile octomap using: make")
MESSAGE (STATUS "Install octomap using: make install")
MESSAGE (STATUS "    (be sure to set the correct CMAKE_INSTALL_PREFIX before)")
MESSAGE (STATUS "Compile API-documentation using: make docs\n")
