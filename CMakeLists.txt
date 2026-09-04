```cmake
cmake_minimum_required(VERSION 3.10)

if(WIN32)
    set(Boost_USE_STATIC_LIBS OFF)

    # The auto-linking feature has problems with USE_STATIC_LIBS off,
    # so disable it and explicitly select the dynamic Boost variants.
    add_definitions(
        -DBOOST_ALL_NO_LIB
        -DBOOST_PROGRAM_OPTIONS_DYN_LINK
        -DBOOST_IOSTREAMS_DYN_LINK
        -DBOOST_THREAD_DYN_LINK
    )
elseif(ASR_BUILD_STATIC_DEPS)
    # ASR_BUILD_STATIC_DEPS means that the ASR project wants its
    # third-party dependencies controlled/bundled by the parent project.
    #
    # Deliberately do NOT touch Boost_USE_STATIC_LIBS here. The parent
    # project sets it (via CACHE FORCE) before calling add_subdirectory() on
    # this file, and setting it again here as a normal variable would shadow
    # that cache value for the rest of this scope, silently undoing whatever
    # the parent asked for.
endif()

project(kenlm)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(FORCE_STATIC "Build static executables" OFF)
option(COMPILE_TESTS "Compile tests" OFF)
option(ENABLE_PYTHON "Build Python bindings" OFF)
option(BUILD_PYTHON_STANDALONE "Build standalone C++ lib with Python install" OFF)

# Eigen3 less than 3.1.0 has a race condition.
find_package(Eigen3 3.1.0 CONFIG)

include(CMakeDependentOption)

cmake_dependent_option(
    ENABLE_INTERPOLATE
    "Build interpolation program (depends on Eigen3)"
    ON
    "EIGEN3_FOUND AND NOT WIN32"
    OFF
)

set(
    KENLM_MAX_ORDER
    6
    CACHE STRING
    "Maximum supported ngram order"
)

if(BUILD_PYTHON_STANDALONE)

    include(
        ${CMAKE_CURRENT_LIST_DIR}/python/BuildStandalone.cmake
    )

    return()

endif()

# ---------------------------------------------------------------------------
# KenLM FORCE_STATIC
#
# This is KenLM's original option and is intentionally separate from
# ASR_BUILD_STATIC_DEPS.
#
# FORCE_STATIC is for KenLM's own static executable builds.
# ASR_BUILD_STATIC_DEPS is for the parent ASR shared-library build.
# ---------------------------------------------------------------------------

if(FORCE_STATIC)

    set(Boost_USE_STATIC_LIBS ON)

    set_property(
        GLOBAL PROPERTY
        LINK_SEARCH_START_STATIC
        ON
    )

    set_property(
        GLOBAL PROPERTY
        LINK_SEARCH_END_STATIC
        ON
    )

    set(
        BUILD_SHARED_LIBRARIES
        OFF
    )

    if(MSVC)

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

                string(
                    REGEX REPLACE
                    "/MD"
                    "/MT"
                    ${flag_var}
                    "${${flag_var}}"
                )

            endif()

        endforeach()

    else()

        if(NOT CMAKE_C_COMPILER_ID MATCHES ".*Clang")

            set(
                CMAKE_EXE_LINKER_FLAGS
                "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++ -static"
            )

        endif()

        set(
            CMAKE_FIND_LIBRARY_SUFFIXES
            ".a"
        )

        # KenLM's original FORCE_STATIC behavior.
        set(CMAKE_EXE_LINK_DYNAMIC_C_FLAGS)
        set(CMAKE_EXE_LINK_DYNAMIC_CXX_FLAGS)
        set(CMAKE_SHARED_LIBRARY_C_FLAGS)
        set(CMAKE_SHARED_LIBRARY_CXX_FLAGS)
        set(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)
        set(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)

    endif()

endif()

# ---------------------------------------------------------------------------
# Output directories
# ---------------------------------------------------------------------------

set(
    EXECUTABLE_OUTPUT_PATH
    ${PROJECT_BINARY_DIR}/bin
)

set(
    LIBRARY_OUTPUT_PATH
    ${PROJECT_BINARY_DIR}/lib
)

if(NOT CMAKE_BUILD_TYPE)

    set(
        CMAKE_BUILD_TYPE
        Release
    )

endif()

# ---------------------------------------------------------------------------
# Tests
# ---------------------------------------------------------------------------

if(COMPILE_TESTS)

    include(CTest)
    enable_testing()

endif()

# ---------------------------------------------------------------------------
# KenLM helper functions
# ---------------------------------------------------------------------------

include(
    cmake/KenLMFunctions.cmake
)

# ---------------------------------------------------------------------------
# MSVC
# ---------------------------------------------------------------------------

if(MSVC)

    set(
        CMAKE_C_FLAGS
        "${CMAKE_C_FLAGS} /w34716"
    )

    set(
        CMAKE_CXX_FLAGS
        "${CMAKE_CXX_FLAGS} /w34716"
    )

endif()

# ---------------------------------------------------------------------------
# KenLM modules
# ---------------------------------------------------------------------------

list(
    APPEND
    CMAKE_MODULE_PATH
    ${PROJECT_SOURCE_DIR}/cmake/modules
)

# ---------------------------------------------------------------------------
# Boost
# ---------------------------------------------------------------------------

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

# ---------------------------------------------------------------------------
# Threads
# ---------------------------------------------------------------------------

set(
    THREADS_PREFER_PTHREAD_FLAG
    ON
)

find_package(
    Threads
    REQUIRED
)

# ---------------------------------------------------------------------------
# KenLM
# ---------------------------------------------------------------------------

add_subdirectory(util)
add_subdirectory(lm)

if(ENABLE_PYTHON)

    add_subdirectory(python)

endif()

# ---------------------------------------------------------------------------
# Installation
# ---------------------------------------------------------------------------

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
        FILES
            ${HEADERS}
        DESTINATION
            include/kenlm/${SUBDIR}
        COMPONENT
            headers
    )

endforeach()

# ---------------------------------------------------------------------------
# Config
# ---------------------------------------------------------------------------

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
```