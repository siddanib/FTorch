cmake_minimum_required(VERSION 3.1 FATAL_ERROR)
#policy CMP0076 - target_sources source files are relative to file where target_sources is run
cmake_policy (SET CMP0076 NEW)

set(PROJECT_NAME MultiIONetExample)

project(${PROJECT_NAME} LANGUAGES Fortran)

# Build in Debug mode if not specified
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug CACHE STRING "" FORCE)
endif()

find_package(FTorch)
message(STATUS "Building with Fortran PyTorch coupling")

# Fortran example
add_executable(multiionet_infer_fortran multiionet_infer_fortran.f90)
target_link_libraries(multiionet_infer_fortran PRIVATE FTorch::ftorch)

# Integration testing
if(CMAKE_BUILD_TESTS)
  include(CTest)

  # 1. Check the PyTorch model runs and its outputs meet expectations
  add_test(NAME multiionet
    COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/multiionet.py)
  set_tests_properties(multiionet PROPERTIES PASS_REGULAR_EXPRESSION
    "0., 2., 4., 6.")
  set_tests_properties(multiionet PROPERTIES PASS_REGULAR_EXPRESSION
    "0., -3., -6., -9.")

  # 2. Check the model is saved to file in the expected location with the pt2ts.py script
  add_test(NAME pt2ts
    COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/pt2ts.py
    ${PROJECT_BINARY_DIR} # Command line argument: filepath for saving the model
    WORKING_DIRECTORY ${PROJECT_BINARY_DIR})

  # 3. Check the model can be loaded from file and run in Python and that its outputs
  #    meet expectations
  add_test(NAME multiionet_infer_python
    COMMAND ${PYTHON_EXECUTABLE} ${PROJECT_SOURCE_DIR}/multiionet_infer_python.py
    ${PROJECT_BINARY_DIR} # Command line argument: filepath to find the model
    WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
  set_tests_properties(multiionet_infer_python PROPERTIES PASS_REGULAR_EXPRESSION
    "0., 2., 4., 6.")
  set_tests_properties(multiionet_infer_python PROPERTIES PASS_REGULAR_EXPRESSION
    "0., -3., -6., -9.")

  # 4. Check the model can be loaded from file and run in Fortran and that its outputs
  #    meet expectations
  add_test(NAME multiionet_infer_fortran
    COMMAND multiionet_infer_fortran
    ${PROJECT_BINARY_DIR}/saved_multiio_model_cpu.pt # Command line argument: model file
    WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
  set_tests_properties(multiionet_infer_fortran PROPERTIES PASS_REGULAR_EXPRESSION
    "   0.00000000       2.00000000       4.00000000       6.00000000")
  set_tests_properties(multiionet_infer_fortran PROPERTIES PASS_REGULAR_EXPRESSION
    "   0.00000000      -3.00000000      -6.00000000      -9.00000000")
endif()
