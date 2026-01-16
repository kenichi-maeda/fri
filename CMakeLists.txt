####################################################################
# cmake support for KUKA's Fast Robot Interface (FRI) client library
####################################################################
cmake_minimum_required(VERSION 3.18)

project(FRIClient VERSION 1.0.0
        DESCRIPTION "KUKA's Fast Robot Interface client library."
        LANGUAGES C CXX)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type." FORCE)
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    # Disable specific warnings (we cannot change these, KUKA has to fix them in their SDK)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-format-security -Wno-parentheses")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-format-security -Wno-parentheses")
endif()

######################
# variable definitions
######################
set(FRIClient_SDK_NAME "FRI-Client-SDK_Cpp")
set(FRI_CLIENT_SDK_VERSIONS "1.11;1.14;1.15;1.16;2.5;2.7" CACHE STRING
    "FRI client SDK versions to build (zip files must exist)")
set(FRI_CLIENT_SDK_DEFAULT_VERSION "1.15" CACHE STRING
    "Default FRI client SDK version for FRIClient::FRIClient")
option(BUILD_FRI_APPS "Build FRIClient example applications for default SDK" OFF)

function(fri_add_sdk version zip_path)
    string(REPLACE "." "_" version_tag "${version}")
    set(sdk_dir "${CMAKE_CURRENT_BINARY_DIR}/${FRIClient_SDK_NAME}_${version_tag}")

    file(ARCHIVE_EXTRACT
        INPUT "${zip_path}"
        DESTINATION "${sdk_dir}"
    )

    set(sdk_root "${sdk_dir}/${FRIClient_SDK_NAME}")
    if(NOT EXISTS "${sdk_root}/include/friClientIf.h")
        set(sdk_root "${sdk_dir}")
    endif()

    file(GLOB NANOPB_HEADER_PATH "${sdk_root}/src/nanopb-*/pb.h")
    if(NOT NANOPB_HEADER_PATH)
        message(FATAL_ERROR "Could not locate nanopb header in ${sdk_root}")
    endif()
    file(READ "${NANOPB_HEADER_PATH}" NANOPB_HEADER)
    string(REGEX MATCH "NANOPB_VERSION nanopb-([0-9]*).([0-9]*).([0-9]*)" _ ${NANOPB_HEADER})

    set(NANOPB_VERSION_MAJOR ${CMAKE_MATCH_1})
    set(NANOPB_VERSION_MINOR ${CMAKE_MATCH_2})
    set(NANOPB_VERSION_PATCH ${CMAKE_MATCH_3})
    set(NANOPB_VERSION ${NANOPB_VERSION_MAJOR}.${NANOPB_VERSION_MINOR}.${NANOPB_VERSION_PATCH})

    file(READ "${sdk_root}/include/friClientIf.h" FRI_CLIENT_IF_HEADER)
    string(REGEX MATCH "version \\{([0-9]+)\\.([0-9]+)" _ ${FRI_CLIENT_IF_HEADER})
    set(FRI_CLIENT_VERSION_MAJOR ${CMAKE_MATCH_1})
    set(FRI_CLIENT_VERSION_MINOR ${CMAKE_MATCH_2})
    set(FRI_CLIENT_VERSION ${FRI_CLIENT_VERSION_MAJOR}.${FRI_CLIENT_VERSION_MINOR})

    if (NOT FRI_CLIENT_VERSION STREQUAL "${version}")
        message(FATAL_ERROR "Expected FRIClient version ${version}, found: ${FRI_CLIENT_VERSION}")
    endif()

    message(STATUS "Configuring FRIClient version ${FRI_CLIENT_VERSION}")
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/friClientVersion.h.in
        ${sdk_root}/include/friClientVersion.h
    )

    file(GLOB_RECURSE NANOPB_SOURCES RELATIVE ${CMAKE_CURRENT_BINARY_DIR}
        ${sdk_root}/src/nanopb-${NANOPB_VERSION}/*.c
    )

    file(GLOB_RECURSE FRI_SOURCES RELATIVE ${CMAKE_CURRENT_BINARY_DIR}
        ${sdk_root}/src/base/*.cpp
        ${sdk_root}/src/client_lbr/*.cpp
        ${sdk_root}/src/client_trafo/*.cpp
        ${sdk_root}/src/connection/*.cpp
        ${sdk_root}/src/protobuf/*.cpp
        ${sdk_root}/src/protobuf/*.c
        ${sdk_root}/src/protobuf_gen/*.pb.c
    )

    add_library(FRIClient_${version_tag} SHARED
        ${FRI_SOURCES}
        ${NANOPB_SOURCES}
    )
    set_target_properties(FRIClient_${version_tag} PROPERTIES
        OUTPUT_NAME "FRIClient_${version_tag}"
    )
    add_library(FRIClient::FRIClient_${version_tag} ALIAS FRIClient_${version_tag})

    target_include_directories(FRIClient_${version_tag}
        PUBLIC
            $<INSTALL_INTERFACE:include/fri/${version}>
            $<BUILD_INTERFACE:${sdk_root}/include>
        PRIVATE
            ${sdk_root}/src/base
            ${sdk_root}/src/nanopb-${NANOPB_VERSION}
            ${sdk_root}/src/protobuf_gen
            ${sdk_root}/src/protobuf
    )

    if(MSVC)
        target_compile_options(FRIClient_${version_tag}
            PRIVATE
                -DPB_SYSTEM_HEADER="pb_syshdr_win.h"
                -DPB_FIELD_16BIT
                -DWIN32
                -DHAVE_STDINT_H
                -DHAVE_STDBOOL_H
        )
    else()
        target_compile_options(FRIClient_${version_tag}
            PRIVATE
                -Wall
                -O2
                -DHAVE_SOCKLEN_T
                -DPB_SYSTEM_HEADER="pb_syshdr.h"
                -DPB_FIELD_16BIT
                -DHAVE_STDINT_H
                -DHAVE_STDDEF_H
                -DHAVE_STDBOOL_H
                -DHAVE_STDLIB_H
                -DHAVE_STRING_H
        )
    endif()

    set(FRI_CLIENT_TARGETS ${FRI_CLIENT_TARGETS} FRIClient_${version_tag} PARENT_SCOPE)
    set(FRI_CLIENT_SDK_ROOT_${version_tag} ${sdk_root} PARENT_SCOPE)
endfunction()

set(FRI_CLIENT_TARGETS "")
foreach(version IN LISTS FRI_CLIENT_SDK_VERSIONS)
    string(REPLACE "." "_" version_tag "${version}")
    set(version_zip "${CMAKE_CURRENT_SOURCE_DIR}/${FRIClient_SDK_NAME}-${version}.zip")
    if (version STREQUAL "${FRI_CLIENT_SDK_DEFAULT_VERSION}")
        if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${FRIClient_SDK_NAME}.zip")
            set(version_zip "${CMAKE_CURRENT_SOURCE_DIR}/${FRIClient_SDK_NAME}.zip")
        endif()
    endif()

    if (EXISTS "${version_zip}")
        message(STATUS "Found SDK zip for ${version}: ${version_zip}")
        fri_add_sdk("${version}" "${version_zip}")
    else()
        message(STATUS "Skipping ${version} (missing ${version_zip})")
    endif()
endforeach()

string(REPLACE "." "_" FRI_CLIENT_DEFAULT_TAG "${FRI_CLIENT_SDK_DEFAULT_VERSION}")
if (TARGET FRIClient_${FRI_CLIENT_DEFAULT_TAG})
    add_library(FRIClient::FRIClient ALIAS FRIClient_${FRI_CLIENT_DEFAULT_TAG})
else()
    message(FATAL_ERROR "Default FRI SDK ${FRI_CLIENT_SDK_DEFAULT_VERSION} not available")
endif()

#################
# install targets
#################
include(CMakePackageConfigHelpers)

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/FRIClientConfigVersion.cmake"
    VERSION ${FRI_CLIENT_SDK_DEFAULT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/FRIClientConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/FRIClientConfig.cmake
    @ONLY
)

install(TARGETS ${FRI_CLIENT_TARGETS}
    EXPORT FRIClientTargets
    LIBRARY DESTINATION lib
    INCLUDES DESTINATION include
)

foreach(version IN LISTS FRI_CLIENT_SDK_VERSIONS)
    string(REPLACE "." "_" version_tag "${version}")
    if (TARGET FRIClient_${version_tag})
        set(root_var "FRI_CLIENT_SDK_ROOT_${version_tag}")
        if(DEFINED ${root_var})
            set(sdk_root "${${root_var}}")
        else()
            set(sdk_root "${CMAKE_CURRENT_BINARY_DIR}/${FRIClient_SDK_NAME}_${version_tag}/${FRIClient_SDK_NAME}")
        endif()
        install(
            DIRECTORY ${sdk_root}/include/
            DESTINATION include/fri/${version}
        )
    endif()
endforeach()

install(EXPORT FRIClientTargets
    FILE FRIClientTargets.cmake
    NAMESPACE FRIClient::
    DESTINATION lib/cmake/FRIClient
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/FRIClientConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/FRIClientConfigVersion.cmake
    DESTINATION lib/cmake/FRIClient
)

################
# build examples
################
if (BUILD_FRI_APPS)
    function(build_fri_example NAME)
        file(GLOB_RECURSE ${NAME}_SOURCES RELATIVE ${CMAKE_CURRENT_BINARY_DIR}
            ${CMAKE_CURRENT_BINARY_DIR}/${FRIClient_SDK_NAME}/example/${NAME}/*.cpp
        )

        add_executable(${NAME}App
            ${${NAME}_SOURCES}
        )

        target_include_directories(${NAME}App
            PRIVATE
                ${CMAKE_CURRENT_BINARY_DIR}/${FRIClient_SDK_NAME}/example/${NAME}
        )

        target_link_libraries(${NAME}App
            PRIVATE
                FRIClient_${FRI_CLIENT_DEFAULT_TAG}
        )
    endfunction()

    build_fri_example(IOAccess)
    build_fri_example(LBRJointSineOverlay)
    build_fri_example(LBRTorqueSineOverlay)
    build_fri_example(LBRWrenchSineOverlay)
    build_fri_example(SimulatedTransformationProvider)
    build_fri_example(TransformationProvider)
endif(BUILD_FRI_APPS)
