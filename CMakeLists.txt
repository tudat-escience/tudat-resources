#    Copyright (c) 2010-2019, Delft University of Technology
#    All rigths reserved
#
#    This file is part of the Tudat. Redistribution and use in source and
#    binary forms, with or without modification, are permitted exclusively
#    under the terms of the Modified BSD license. You should have received
#    a copy of the license with this file. If not, please or visit:
#    http://tudat.tudelft.nl/LICENSE.

# Specify minimum cmake version required to compile Tudat.
cmake_minimum_required(VERSION 3.5)

# Get version from file.
file(STRINGS "version" TudatResources_VERSION)

# Specific project name.
project(TudatResources VERSION ${TudatResources_VERSION} LANGUAGES C CXX)

# Get lower string name of PROJECT_NAME for directory configuration.
string(TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWER)

# +============================================================================
# INSTALL TREE CONFIGURATION (Project name independent)
#  Offer the user the choice of overriding the installation directories.
# +============================================================================
set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib/" CACHE PATH "Installation directory for libraries")
set(INSTALL_INCLUDE_DIR "${CMAKE_INSTALL_PREFIX}/include/" CACHE PATH "Installation directory for header files")
set(INSTALL_CMAKE_DIR "${INSTALL_LIB_DIR}/cmake/${PROJECT_NAME_LOWER}" CACHE PATH "Installation directory for cmake config files")

if (NOT IS_ABSOLUTE "${INSTALL_CMAKE_DIR}")
    set(INSTALL_CMAKE_DIR "${CMAKE_INSTALL_PREFIX}/${INSTALL_CMAKE_DIR}")
endif ()

# +============================================================================
# INSTALL TREE
#
# +============================================================================
add_executable(test_header test/test_header.cpp)

set_target_properties(test_header
        PROPERTIES
        LINKER_LANGUAGE CXX
        ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib"
        LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib"
        RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/tests"
        )

target_include_directories(test_header PUBLIC
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:include>)

enable_testing()

add_test(test_header tests/test_header)
file(COPY include DESTINATION ${PROJECT_BINARY_DIR})

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/ DESTINATION "${INSTALL_INCLUDE_DIR}")
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/test/test_binary/ DESTINATION "${CMAKE_INSTALL_PREFIX}/tests")
# install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/resource/ DESTINATION "${INSTALL_DATA_DIR}")

# CONFIGURE
configure_file( # Really dirty trick, and should be removed.
        "${PROJECT_SOURCE_DIR}/config.hpp.in"
        "${PROJECT_BINARY_DIR}/include/tudat/resource/config.hpp" @ONLY
        )

install(FILES
        "${PROJECT_BINARY_DIR}/include/tudat/resource/config.hpp"
        DESTINATION ${INSTALL_INCLUDE_DIR}/tudat/resource)

# Installation of the library.
install(TARGETS test_header
        EXPORT tudatresources_export
        LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
        ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
        RUNTIME DESTINATION "${CMAKE_INSTALL_PREFIX}/tests/"
        )

# Add all targets to the build-tree export set
export(TARGETS test_header NAMESPACE ${PROJECT_NAME}:: FILE "${CMAKE_CURRENT_BINARY_DIR}/lib/cmake/${PROJECT_NAME_LOWER}/${PROJECT_NAME_LOWER}_export.cmake")

# Installation of the export modules.
install(EXPORT ${PROJECT_NAME_LOWER}_export NAMESPACE ${PROJECT_NAME}:: DESTINATION "${RELATIVE_INSTALL_CMAKE_DIR}")

# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
set(CMAKE_EXPORT_PACKAGE_REGISTRY ON)
export(PACKAGE ${PROJECT_NAME})

# +============================================================================
# PROJECT CONFIG
#  For external use of project.
# +============================================================================
include(CMakePackageConfigHelpers)

configure_package_config_file(
        "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME_LOWER}-config.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/lib/cmake/${PROJECT_NAME_LOWER}/${PROJECT_NAME_LOWER}-config.cmake"
        INSTALL_DESTINATION "${INSTALL_CMAKE_DIR}")

install(
        FILES
        "${CMAKE_CURRENT_BINARY_DIR}/lib/cmake/${PROJECT_NAME_LOWER}/${PROJECT_NAME_LOWER}-config.cmake"
        DESTINATION "${INSTALL_CMAKE_DIR}"
)

# NOTE: SameMinorVersion available only
# since CMake 3.11.
if (${CMAKE_VERSION} VERSION_LESS "3.11.0")
    write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME_LOWER}-config-version.cmake" VERSION ${${PROJECT_NAME}_VERSION}
            COMPATIBILITY SameMajorVersion)
else ()
    write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME_LOWER}-config-version.cmake" VERSION ${${PROJECT_NAME}_VERSION}
            COMPATIBILITY SameMinorVersion)
endif ()
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME_LOWER}-config-version.cmake" DESTINATION "${INSTALL_CMAKE_DIR}")

# +============================================================================
# CLEAN UP (Project name independent)
#  Cleanup in case project is not top level.
# +============================================================================
unset(INSTALL_LIB_DIR)
unset(INSTALL_INCLUDE_DIR)
unset(INSTALL_CMAKE_DIR)
