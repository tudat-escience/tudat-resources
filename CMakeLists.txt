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
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if(NOT PROJECT_IS_TOP_LEVEL)
  # used to resolve resource path
  target_compile_definitions(tudat_resources INTERFACE SOURCE_BUILD)
endif()
