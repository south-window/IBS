cmake_minimum_required(VERSION 3.10.2)

SET(CMAKE_CXX_STANDARD 17)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_CXX_EXTENSIONS OFF)
SET(GIT_SUBMODULE ON)

# setting build type if none is given
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# setting compiler flags
SET(GCC_OPENMP_COMPILE_FLAGS "-fopenmp -D use_openmp -Ofast -march=native -ffast-math")
SET(GCC_OPENMP_LINK_FLAGS "-fopenmp ")

SET(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${GCC_OPENMP_COMPILE_FLAGS}")
SET(CMAKE_CXX_FLAGS_RELEASE  "${CMAKE_CXX_FLAGS} ${GCC_OPENMP_COMPILE_FLAGS}")
SET(CMAKE_EXE_LINKER_FLAGS  "${CMAKE_EXE_LINKER_FLAGS} ${GCC_OPENMP_LINK_FLAGS}")


PROJECT(IBSLib VERSION 0.2.0)

# DOWNLOAD THE GIT SUBMODULES (PYBIND11 HERE)
find_package(Git QUIET)

if (GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
    # update as needed
    option(GIT_SUBMODULE "Check submodules during build" ON)

    if(GIT_SUBMODULE)
        message(STATUS "Updating submodules...")
        execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            RESULT_VARIABLE GIT_SUBMOD_RESULT)
        if(NOT GIT_SUBMOD_RESULT EQUAL "0")
            message(FATAL_ERROR "git submodule update --init failed with ${GIT_SUBMOD_RESULT}, please check...")
        endif()
    endif()
endif()

# CHECK ALL THE SUBMODULES
if(NOT EXISTS "${PROJECT_SOURCE_DIR}/third_party/pybind11-2.6.2/CMakeLists.txt")
    message(FATAL_ERROR "The PYBIND11 submodule was not downloaded! GIT_SUBMODULE was turned off or failed.")
endif()

# find python
find_program(
    PYTHON_EXECUTABLE
    NAMES python
)

# if python found 
# get the PYTHON_LIBRARY_DIR to allow to install files
if(PYTHON_EXECUTABLE)
    message(STATUS "PYTHON_EXECUTABLE = ${PYTHON_EXECUTABLE}")

    execute_process(
        COMMAND "${PYTHON_EXECUTABLE}" -c
        "from distutils import sysconfig;print(sysconfig.get_config_var('LIBDIR'))"
            OUTPUT_VARIABLE PYTHON_LIBRARY_DIR
            RESULT_VARIABLE PYTHON_LIBRARY_DIR_RESULT
            ERROR_QUIET)    
    string(STRIP ${PYTHON_LIBRARY_DIR} PYTHON_LIBRARY_DIR)
    if(NOT PYTHON_LIBRARY_DIR_RESULT MATCHES 0)
        message(SEND_ERROR "Failed to determine PYTHON_LIBRARY")
    endif()
    execute_process(
        COMMAND "${PYTHON_EXECUTABLE}" -c "import site; print(site.getsitepackages()[0])"
        OUTPUT_VARIABLE _site_packages
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
else()
    message(FATAL_ERROR "python executable not found.")
endif()


# adding pybind11
# git submodule add https://github.com/pybind/pybind11.git third_party/pybind11-2.5.0
SET(MY_PYBIND ${CMAKE_CURRENT_SOURCE_DIR}/third_party/pybind11-2.6.2)
add_subdirectory(${MY_PYBIND})


include_directories("${CMAKE_SOURCE_DIR}/cpp/include/ibs_bits")
include_directories("${CMAKE_SOURCE_DIR}/python")

file (GLOB SOURCE_FILES "cpp/src/*.cpp")
file (GLOB HEADER_FILES "cpp/include/ibs_bits/*.hpp")
file (GLOB PYTHON_FILES "python/*.cpp" "python/*.hpp")

message(STATUS "SOURCE_FILES = ${SOURCE_FILES}")
message(STATUS "HEADER_FILES = ${HEADER_FILES}")
message(STATUS "PYTHON_FILES = ${PYTHON_FILES}")

# Set up such that XCode organizes the files
source_group(TREE ${CMAKE_CURRENT_SOURCE_DIR} FILES ${SOURCE_FILES} ${HEADER_FILES} ${PYTHON_FILES} )

# include(pybind11.cmake)
pybind11_add_module(${PROJECT_NAME} 
${SOURCE_FILES}
${HEADER_FILES}
${PYTHON_FILES}
)


target_link_libraries(${PROJECT_NAME}  PUBLIC)

if(NOT _site_packages )
    set(_site_packages  ${PYTHON_LIBRARY_DIR})
endif()

# show the PYTHON_LIBRARY_DIR
message(STATUS "PYTHON_LIBRARY_DIR = ${PYTHON_LIBRARY_DIR}")
message(STATUS "_site_package      = ${_site_packages}")

install(TARGETS ${PROJECT_NAME} 
COMPONENT python
#LIBRARY DESTINATION "${PYTHON_LIBRARY_DIR}"
LIBRARY DESTINATION "${_site_packages}"
)
