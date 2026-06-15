#    Copyright (c) 2010-2019, Delft University of Technology
#    All rigths reserved
#
#    This file is part of the Tudat. Redistribution and use in source and
#    binary forms, with or without modification, are permitted exclusively
#    under the terms of the Modified BSD license. You should have received
#    a copy of the license with this file. If not, please or visit:
#    http://tudat.tudelft.nl/LICENSE.

cmake_minimum_required(VERSION 3.21)
# Get version from file.
file(STRINGS "version" TudatResources_VERSION)
project(TudatResources VERSION ${TudatResources_VERSION} LANGUAGES C CXX)

include(GNUInstallDirs)

add_library(tudat_resources INTERFACE)
add_library(tudat_resources::tudat_resources ALIAS tudat_resources)

# doesn't need to be installed, only relevant when built from source
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/config.hpp.in"
    "${CMAKE_CURRENT_BINARY_DIR}/include/tudat/resource/config.hpp"
    @ONLY
)

target_include_directories(tudat_resources INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include/tudat/resource>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if(NOT PROJECT_IS_TOP_LEVEL)
  # used to resolve resource path
  target_compile_definitions(tudat_resources INTERFACE SOURCE_BUILD)
endif()

if(PROJECT_IS_TOP_LEVEL)
  # Tests
  include(CTest)
  add_executable(test_header test/test_header.cpp)
  target_link_libraries(test_header PRIVATE tudat_resources)
  target_compile_definitions(test_header PRIVATE SOURCE_BUILD)

  add_test(NAME test_header COMMAND test_header)

  # CMake package configuration
  include(CMakePackageConfigHelpers)

  install(TARGETS tudat_resources EXPORT tudat_resources-targets
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )

  # rsync rule for trailing slash in directory names
  install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/tudat
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  )
  install(EXPORT tudat_resources-targets
    FILE tudat_resources-targets.cmake
    NAMESPACE tudat_resources::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/tudat_resources
  )

  configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/tudat_resources-config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/tudat_resources-config.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/tudat_resources
  )

  write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/tudat_resources-config-version.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMinorVersion
  )

  install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/tudat_resources-config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/tudat_resources-config-version.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/tudat_resources
  )
endif()
