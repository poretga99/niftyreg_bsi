
cmake_minimum_required(VERSION 3.18.0)
project(NiftyReg LANGUAGES C CXX CUDA)
#-----------------------------------------------------------------------------
set(NiftyReg_VERSION_MAJOR 1)
set(NiftyReg_VERSION_MINOR 3)
set(NiftyReg_VERSION_PATCH 9)
#-----------------------------------------------------------------------------
option(BUILD_NR_DEV "To build NiftyReg code under development" OFF)
option(BUILD_TESTING "To build the unit tests" OFF)
option(BUILD_NR_SLICER_EXT "Where NiftyReg meets 3DSlicer" OFF)
option(BUILD_ALL_DEP "All the dependencies are build" OFF)
# All dependencies are build to create the 3DSlicer package
if(BUILD_NR_SLICER_EXT)
    set(BUILD_ALL_DEP ON)
    option(BUILD_NR_DEV ON)
    mark_as_advanced(FORCE BUILD_ALL_DEP)
    mark_as_advanced(FORCE BUILD_NR_DEV)
else(BUILD_NR_SLICER_EXT)
    mark_as_advanced(CLEAR BUILD_ALL_DEP)
    mark_as_advanced(CLEAR BUILD_NR_DEV)
endif(BUILD_NR_SLICER_EXT)

option(USE_SSE "To enable SEE computation in some case" OFF)
option(USE_CUDA "To enable CUDA for a GPU implementation of the code" ON)
option(USE_DOUBLE "To force double precision instead of single precision" OFF)
if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    # I removed openmp for Mac OS Lion
    if(NOT CMAKE_SYSTEM_VERSION VERSION_GREATER 11.0 OR CMAKE_SYSTEM_VERSION VERSION_EQUAL 11.0.0)
        option(USE_OPENMP "To use openMP for multi-CPU processing" ON)
    endif()
endif()
#-----------------------------------------------------------------------------
include_directories(${CMAKE_SOURCE_DIR}/reg-lib)
include_directories(${CMAKE_SOURCE_DIR}/reg-io)
include_directories(${CMAKE_SOURCE_DIR}/reg-io/nifti)
#-----------------------------------------------------------------------------
# Z library
# Try first to find the z library on the system and built is from the sources if it can not be find
if(NOT BUILD_ALL_DEP)
    find_package(ZLIB)
    if(ZLIB_FOUND)
        include_directories(${ZLIB_INCLUDE_DIR})
            message(STATUS "Found zlib - the z library will not be built")
    else(ZLIB_FOUND)
        include_directories(${CMAKE_SOURCE_DIR}/reg-io/zlib)
        message(STATUS "zlib not found - the z library will be built")
    endif(ZLIB_FOUND)
else(NOT BUILD_ALL_DEP)
    include_directories(${CMAKE_SOURCE_DIR}/reg-io/zlib)
endif(NOT BUILD_ALL_DEP)
#-----------------------------------------------------------------------------
## PNG support - First try to find the PNG library on the system and build it if it is not found
## I did not use the FindPNG.cmake here as the zlib is also included into the project

# Try to find the png library and header on the system
if(CYGWIN)
    if(NOT BUILD_SHARED_LIBS)
        set (PNG_DEFINITIONS -DPNG_STATIC)
    endif(NOT BUILD_SHARED_LIBS)
endif(CYGWIN)
if(NOT BUILD_ALL_DEP)
    set(PNG_NAMES ${PNG_NAMES} png libpng png15 libpng15 png15d libpng15d png14 libpng14 png14d libpng14d png12 libpng12 png12d libpng12d)
    find_library(PNG_LIBRARY NAMES ${PNG_NAMES})
    find_path(PNG_INCLUDE_DIR png.h
        /usr/local/include/libpng
        /sw/include
    )
    # If the png library and header can not be found, it is build from the sources
    if(NOT PNG_LIBRARY OR NOT PNG_INCLUDE_DIR)
        message(STATUS "libpng not found - the png library will be built")
        set(PNG_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/reg-io/png/lpng1510)
        set(PNG_LIBRARY png)
        set(BUILD_INTERNAL_PNG true)
    else(NOT PNG_LIBRARY OR NOT PNG_INCLUDE_DIR)
        message(STATUS "Found libpng - the png library will not be built")
        set(BUILD_INTERNAL_PNG false)
    endif(NOT PNG_LIBRARY OR NOT PNG_INCLUDE_DIR)
else(NOT BUILD_ALL_DEP)
    set(PNG_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/reg-io/png/lpng1510)
    set(PNG_LIBRARY png)
endif(NOT BUILD_ALL_DEP)
include_directories(${CMAKE_SOURCE_DIR}/reg-io/png)
include_directories(${PNG_INCLUDE_DIR})


#-----------------------------------------------------------------------------
# NRRD file format support - The nrrd file format has been embedded into the project
include_directories(${CMAKE_BINARY_DIR})
include_directories(${CMAKE_SOURCE_DIR}/reg-io/nrrd)
include_directories(${CMAKE_SOURCE_DIR}/reg-io/nrrd/NrrdIO)

#-----------------------------------------------------------------------------
# Double can not be used with SSE as it has only been implemented for single precision
# Double can not be used with CUDA
if(USE_DOUBLE)
    add_definitions(-D_USE_NR_DOUBLE)
    if(USE_SSE)
        message("Can not use double precision and SSE implementation concurrenlty")
        message(FATAL_ERROR "Please turn USE_SSE OFF to use double precision")
    endif(USE_SSE)
    if(USE_CUDA)
        message("Can not use double precision and CUDA implementation concurrenlty")
        message(FATAL_ERROR "Please turn USE_CUDA OFF to use double precision")
    endif(USE_CUDA)
endif(USE_DOUBLE)

if(USE_SSE)
    add_definitions(-D_USE_SSE)
    if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=native")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
    endif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
endif(USE_SSE)

if(BUILD_NR_DEV)
    add_definitions(-D_BUILD_NR_DEV)
endif(BUILD_NR_DEV)

if(USE_OPENMP)
    if(GCC_VERSION VERSION_GREATER 4.2 OR GCC_VERSION VERSION_EQUAL 4.2)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fopenmp")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")
    endif(GCC_VERSION VERSION_GREATER 4.2 OR GCC_VERSION VERSION_EQUAL 4.2)
	
    if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /openmp")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /openmp")
    endif(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
endif(USE_OPENMP)

#-----------------------------------------------------------------------------
if(USE_CUDA)
    find_package(CUDAToolkit REQUIRED)
    include_directories(${CUDAToolkit_INCLUDE_DIRS})
endif(USE_CUDA)
#-----------------------------------------------------------------------------
if(IS_DIRECTORY ${CMAKE_SOURCE_DIR}/.svn)
    find_package(Subversion)
    if(Subversion_FOUND)
        Subversion_WC_INFO(${CMAKE_SOURCE_DIR} NiftyReg)
        add_definitions(-D_SVN_REV=${NiftyReg_WC_REVISION})
        message(STATUS "Found subversion - the revision number is added to the source.")
    endif(Subversion_FOUND)
endif(IS_DIRECTORY ${CMAKE_SOURCE_DIR}/.svn)
#-----------------------------------------------------------------------------
add_subdirectory(reg-io)
add_subdirectory(reg-lib)
add_subdirectory(reg-apps)