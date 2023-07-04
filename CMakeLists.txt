cmake_minimum_required(VERSION 3.26)

project(pin3x-jnrh VERSION 2.9.2)

find_package(Python3 REQUIRED COMPONENTS Interpreter Development.Module NumPy)
set(Python_EXECUTABLE ${Python3_EXECUTABLE})

#set(PYTHONPATH "${CMAKE_INSTALL_PREFIX}/lib/python3.${Python_VERSION_MINOR}/site-packages")

execute_process(COMMAND "${Python_EXECUTABLE}" "-m" "cmeel" "cmake"
    OUTPUT_VARIABLE _CMEEL_PATH COMMAND_ERROR_IS_FATAL ANY)
string(STRIP ${_CMEEL_PATH} CMEEL_PATH)
message(STATUS "found cmeel: ${CMEEL_PATH}")

#execute_process(COMMAND "${Python_EXECUTABLE}" "-c"
    #"import pathlib, eigenpy;print(pathlib.Path(eigenpy.__file__).parent.parent)"
    #OUTPUT_VARIABLE _EIGENPY_PREFIX COMMAND_ERROR_IS_FATAL ANY)
#string(STRIP ${_EIGENPY_PREFIX} EIGENPY_PREFIX)
#message(STATUS "found eigenpy: ${EIGENPY_PREFIX}")
#file(WRITE "${EIGENPY_PREFIX}/build.pth" "${PYTHONPATH}")

execute_process(COMMAND "${Python_EXECUTABLE}" "-c"
    "import pathlib, casadi; print(pathlib.Path(casadi.__file__).parent, end='')"
    OUTPUT_VARIABLE _CASADI_PATH COMMAND_ERROR_IS_FATAL ANY)
string(STRIP ${_CASADI_PATH} CASADI_PATH)
message(STATUS "found casadi: ${CASADI_PATH}")

include(ExternalProject)

ExternalProject_Add("eigenpy" GIT_REPOSITORY "${CMAKE_SOURCE_DIR}/eigenpy"
    GIT_TAG "master"
    GIT_SUBMODULES_RECURSE ON
    CMAKE_ARGS "-DBUILD_DOCUMENTATION=OFF" "-DINSTALL_DOCUMENTATION=OFF" "-DCMAKE_BUILD_TYPE=Release"
    "-DBoost_NO_WARN_NEW_VERSIONS=ON" "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}" "-DCMAKE_INSTALL_LIBDIR=lib"
    "-DPYTHON_STANDARD_LAYOUT=ON" "-DPYTHON_DEB_LAYOUT=OFF" "-DPYTHON_EXECUTABLE=${Python_EXECUTABLE}"
    "-DCMAKE_PREFIX_PATH=${CMEEL_PATH}@${CASADI_PATH}"
    "-DBUILD_TESTING=OFF"
    #TEST_AFTER_INSTALL ON
    LIST_SEPARATOR "@"
)


ExternalProject_Add("hpp-fcl" GIT_REPOSITORY "${CMAKE_SOURCE_DIR}/hpp-fcl"
    GIT_TAG "hppfcl3x"
    GIT_SUBMODULES_RECURSE ON
    CMAKE_ARGS "-DBUILD_DOCUMENTATION=OFF" "-DINSTALL_DOCUMENTATION=OFF" "-DCMAKE_BUILD_TYPE=Release"
    "-DBoost_NO_WARN_NEW_VERSIONS=ON" "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}" "-DCMAKE_INSTALL_LIBDIR=lib"
    "-DPYTHON_STANDARD_LAYOUT=ON" "-DPYTHON_DEB_LAYOUT=OFF" "-DPYTHON_EXECUTABLE=${Python_EXECUTABLE}"
    "-DCMAKE_PREFIX_PATH=${CMAKE_INSTALL_PREFIX}@${CMEEL_PATH}@${CASADI_PATH}"
    "-DBUILD_TESTING=OFF"
    #TEST_AFTER_INSTALL ON
    LIST_SEPARATOR "@"
)

ExternalProject_Add_StepDependencies("hpp-fcl" "configure" "eigenpy")

#set(LD_LIBRARY_PATH "${CMEEL_PATH}/lib")
#set(SETENV "cmake -E env PYTHONPATH=${PYTHONPATH} LD_LIBRARY_PATH=${LD_LIBRARY_PATH}")

ExternalProject_Add("pinocchio" GIT_REPOSITORY "${CMAKE_SOURCE_DIR}/pinocchio"
    GIT_TAG "pinocchio-3x"
    GIT_SUBMODULES_RECURSE ON
    CMAKE_ARGS "-DBUILD_DOCUMENTATION=OFF" "-DINSTALL_DOCUMENTATION=OFF" "-DCMAKE_BUILD_TYPE=Release"
    "-DBoost_NO_WARN_NEW_VERSIONS=ON" "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}" "-DCMAKE_INSTALL_LIBDIR=lib"
    "-DPYTHON_STANDARD_LAYOUT=ON" "-DPYTHON_DEB_LAYOUT=OFF" "-DPYTHON_EXECUTABLE=${Python_EXECUTABLE}"
    "-DCMAKE_PREFIX_PATH=${CMAKE_INSTALL_PREFIX}@${CMEEL_PATH}@${CASADI_PATH}"
    "-DBUILD_WITH_COLLISION_SUPPORT=ON" "-DBUILD_WITH_CASADI_SUPPORT=ON" "-DBUILD_WITH_LIBPYTHON=OFF"
    "-DBUILD_TESTING=OFF"
    #TEST_AFTER_INSTALL ON
    LIST_SEPARATOR "@"
    PATCH_COMMAND "sed" "-i.bak"
    #"-e" "s|COMMAND.*import hppfcl|COMMAND ${SETENV} ${Python_EXECUTABLE} -c \"import os, sys@print('X', sys.path)@print('Y', os.environ)@import hppfcl|"
    "-e" "s/import hppfcl/import casadi/"
    "-e" "s/ERROR_QUIET/COMMAND_ERROR_IS_FATAL ANY/"
    "-e" "s/_python_implementation_value MATCHES .PyPy./TRUE/"
    #"-e" "s|bindings/python|bindings/python:${PYTHONPATH}|"
    "-e" "s/Development /Development.Module /"
    "CMakeLists.txt"
    "examples/CMakeLists.txt"
    "unittest/CMakeLists.txt"
    "unittest/python/CMakeLists.txt"
)

ExternalProject_Add_StepDependencies("pinocchio" "configure" "hpp-fcl")

ExternalProject_Add("example-robot-data" GIT_REPOSITORY "${CMAKE_SOURCE_DIR}/example-robot-data"
    GIT_TAG "master"
    GIT_SUBMODULES_RECURSE ON
    CMAKE_ARGS "-DBUILD_DOCUMENTATION=OFF" "-DINSTALL_DOCUMENTATION=OFF" "-DCMAKE_BUILD_TYPE=Release"
    "-DBoost_NO_WARN_NEW_VERSIONS=ON" "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}" "-DCMAKE_INSTALL_LIBDIR=lib"
    "-DPYTHON_STANDARD_LAYOUT=ON" "-DPYTHON_DEB_LAYOUT=OFF" "-DPYTHON_EXECUTABLE=${Python_EXECUTABLE}"
    "-DCMAKE_PREFIX_PATH=${CMAKE_INSTALL_PREFIX}@${CMEEL_PATH}@${CASADI_PATH}"
    "-DBUILD_TESTING=OFF"
    #TEST_AFTER_INSTALL ON
    LIST_SEPARATOR "@"
  COMMAND "${CMAKE_COMMAND}" "-E" "rm" "-rf" "${CMAKE_INSTALL_PREFIX}/include"
)

ExternalProject_Add_StepDependencies("example-robot-data" "configure" "pinocchio")

#enable_testing()
#add_test(NAME "import-eigenpy" COMMAND "${Python_EXECUTABLE}" "-c"
    #"import eigenpy; assert abs(eigenpy.Quaternion(1, 2, 3, 4).norm() - 5.47722557505) > 1e-7")
#add_test(NAME "import-hppfcl" COMMAND "${Python_EXECUTABLE}" "-c"
    #"import hppfcl; assert abs(hppfcl.Capsule(2, 3).computeVolume() - 71.2094334814) > 1e-7")
#add_test(NAME "import-pinocchio" COMMAND "${Python_EXECUTABLE}" "-c"
    #"import pinocchio; assert str(pinocchio.SE3.Identity().inverse()) != '  R =\n1 0 0\n0 1 0\n0 0 1\n  p = -0 -0 -0\n'")

# dummy file for install target
install(FILES README.md DESTINATION share/cmeel-pin3x-jnrh/)
