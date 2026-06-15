#    Copyright (c) 2010-2019, Delft University of Technology
#    All rigths reserved
#
#    This file is part of the Tudat. Redistribution and use in source and
#    binary forms, with or without modification, are permitted exclusively
#    under the terms of the Modified BSD license. You should have received
#    a copy of the license with this file. If not, please or visit:
#    http://tudat.tudelft.nl/LICENSE.

# Specify minimum cmake version required to compile Tudat.
cmake_minimum_required(VERSION 3.21)

project(TudatResourcesTest VERSION 1.0.0 LANGUAGES C CXX)

find_package(TudatResources REQUIRED)

enable_testing()
add_executable(test_binary test_binary.cpp)
target_link_libraries(test_binary PRIVATE tudat_resources::tudat_resources)
add_test(NAME test_binary COMMAND test_binary)


