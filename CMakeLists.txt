cmake_minimum_required(VERSION 3.10)

# Define a single cmake project
project(kenlm)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ------------------------------------------------------------------
# KenLM build options
# ------------------------------------------------------------------

option(FORCE_STATIC "Build static executables" OFF)
option(COMPILE_TESTS "Compile tests" OFF)
option(ENABLE_PYTHON "Build Python bindings" OFF)
option(BUILD_PYTHON_STANDALONE "Build standalone C++ lib with Python install" OFF)

# KenLM itself is always built as static libraries when included by
# the parent ASR project.
#
# The parent project sets BUILD_SHARED_LIBS=OFF, but explicitly set
# it here as well so KenLM cannot accidentally produce shared libs.
set(BUILD_SHARED_LIBS OFF)

# ------------------------------------------------------------------
# Static Boost
#
# KenLM needs Boost libraries, but they must be linked statically
# into the ASR shared library.
# ------------------------------------------------------------------

set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_STATIC_RUNTIME ON)
set(Boost_USE_MULTITHREADED ON)

# ------------------------------------------------------------------
# Eigen3
# ------------------------------------------------------------------

find_package(Eigen3 3.1.0 CONFIG)

include(CMakeDependentOption)

cmake_dependent_option(
    ENABLE_INTERPOLATE
    "Build interpolation program (depends on Eigen3)"
    ON
    "EIGEN3_FOUND AND NOT WIN32"
    OFF
)

set(KENLM_MAX_ORDER 6 CACHE STRING
    "Maximum supported ngram order"
)

# ------------------------------------------------------------------
# Python standalone build
# ------------------------------------------------------------------

if (BUILD_PYTHON_STANDALONE)

    include(
        ${CMAKE_CURRENT_LIST_DIR}/python/BuildStandalone.cmake
    )

    return()

endif()

# ------------------------------------------------------------------
# FORCE_STATIC
#
# This option is retained for KenLM compatibility.
#
# It is NOT enabled by the ASR project because the ASR library is
# shared and only its third-party libraries need to be static.
# ------------------------------------------------------------------

if (FORCE_STATIC OR ASR_BUILD_STATIC_DEPS)
    set(Boost_USE_STATIC_LIBS ON)
    set(Boost_USE_STATIC_RUNTIME ON)

    set_property(GLOBAL PROPERTY LINK_SEARCH_START_STATIC ON)
    set_property(GLOBAL PROPERTY LINK_SEARCH_END_STATIC ON)
    set(BUILD_SHARED_LIBRARIES OFF)

    if (MSVC)
        set(flag_vars
            CMAKE_CXX_FLAGS
            CMAKE_CXX_FLAGS_DEBUG
            CMAKE_CXX_FLAGS_RELEASE
            CMAKE_CXX_FLAGS_MINSIZEREL
            CMAKE_CXX_FLAGS_RELWITHDEBINFO
            CMAKE_C_FLAGS
            CMAKE_C_FLAGS_DEBUG
            CMAKE_C_FLAGS_RELEASE
            CMAKE_C_FLAGS_MINSIZEREL
            CMAKE_C_FLAGS_RELWITHDEBINFO
        )

        foreach(flag_var ${flag_vars})
            if(${flag_var} MATCHES "/MD")
                string(REGEX REPLACE "/MD" "/MT"
                    ${flag_var} "${${flag_var}}")
            endif()
        endforeach()
    else()
        if (NOT CMAKE_C_COMPILER_ID MATCHES ".*Clang")
            set(CMAKE_EXE_LINKER_FLAGS
                "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++"
            )
        endif()
    endif()
endif()

# ------------------------------------------------------------------
# Output directories
# ------------------------------------------------------------------

set(
    EXECUTABLE_OUTPUT_PATH
    ${PROJECT_BINARY_DIR}/bin
)

set(
    LIBRARY_OUTPUT_PATH
    ${PROJECT_BINARY_DIR}/lib
)

if (NOT CMAKE_BUILD_TYPE)

    set(CMAKE_BUILD_TYPE Release)

endif()

# ------------------------------------------------------------------
# Tests
# ------------------------------------------------------------------

if (COMPILE_TESTS)

    include(CTest)
    enable_testing()

endif()

# ------------------------------------------------------------------
# KenLM helper functions
# ------------------------------------------------------------------

include(
    cmake/KenLMFunctions.cmake
)

# ------------------------------------------------------------------
# MSVC
# ------------------------------------------------------------------

if(MSVC)

    set(
        CMAKE_C_FLAGS
        "${CMAKE_CXX_FLAGS} /w34716"
    )

    set(
        CMAKE_CXX_FLAGS
        "${CMAKE_CXX_FLAGS} /w34716"
    )

endif()

# ------------------------------------------------------------------
# KenLM modules
# ------------------------------------------------------------------

list(
    APPEND
    CMAKE_MODULE_PATH
    ${PROJECT_SOURCE_DIR}/cmake/modules
)

# ------------------------------------------------------------------
# Boost
#
# IMPORTANT:
# Boost_USE_STATIC_LIBS is ON above.
#
# Do NOT use the Windows dynamic-link definitions from the original
# KenLM CMakeLists.txt.
# ------------------------------------------------------------------

find_package(
    Boost 1.41.0
    REQUIRED
    COMPONENTS
        program_options
        system
        thread
        unit_test_framework
)

include_directories(
    ${Boost_INCLUDE_DIRS}
)

# ------------------------------------------------------------------
# Threads
# ------------------------------------------------------------------

set(
    THREADS_PREFER_PTHREAD_FLAG
    ON
)

find_package(
    Threads
    REQUIRED
)

# ------------------------------------------------------------------
# KenLM source
# ------------------------------------------------------------------

add_subdirectory(util)
add_subdirectory(lm)

if(ENABLE_PYTHON)

    add_subdirectory(python)

endif()

# ------------------------------------------------------------------
# Installation
# ------------------------------------------------------------------

install(
    EXPORT kenlmTargets
    FILE kenlmTargets.cmake
    NAMESPACE kenlm::
    DESTINATION share/kenlm/cmake
)

foreach(
    SUBDIR
    IN ITEMS
        util
        util/double-conversion
        util/stream
        lm
        lm/builder
        lm/common
        lm/filter
        lm/interpolate
)

    file(
        GLOB
        HEADERS
            ${CMAKE_CURRENT_LIST_DIR}/${SUBDIR}/*.h
            ${CMAKE_CURRENT_LIST_DIR}/${SUBDIR}/*.hh
    )

    install(
        FILES ${HEADERS}
        DESTINATION include/kenlm/${SUBDIR}
        COMPONENT headers
    )

endforeach()

# ------------------------------------------------------------------
# Config
# ------------------------------------------------------------------

include(
    CMakePackageConfigHelpers
)

configure_package_config_file(
    ${PROJECT_SOURCE_DIR}/cmake/kenlmConfig.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/kenlmConfig.cmake"
    INSTALL_DESTINATION share/kenlm/cmake
    NO_SET_AND_CHECK_MACRO
    NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

install(
    FILES
        ${CMAKE_CURRENT_BINARY_DIR}/kenlmConfig.cmake
    DESTINATION
        share/kenlm/cmake
)
