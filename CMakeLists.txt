cmake_minimum_required(VERSION 3.23)

project(tdscpp VERSION 20250301)

set(libname "tdscpp")

###############
# ASI default settings
list(APPEND CMAKE_MODULE_PATH "$ENV{ASV_CMAKE}")
include(asv_cmake_defaults)
asv_cmake_defaults()
###############

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
include(CheckIncludeFileCXX)

option(BUILD_SAMPLE        "Build sample program" OFF)
option(WITH_OPENSSL        "Use OpenSSL for encryption" ON)
option(ENABLE_KRB5         "Fuck" OFF)

if(NOT WIN32)
    option(ENABLE_KRB5         "Enable Kerberos support" ON)
endif()

find_package(nlohmann_json CONFIG REQUIRED)
asv_print_package_properties(
    nlohmann_json::nlohmann_json
)

if(ENABLE_KRB5)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(KRB5 REQUIRED IMPORTED_TARGET mit-krb5-gssapi)
    set(HAVE_GSSAPI 1)
endif()

if(NOT WIN32)
    find_package(ICU COMPONENTS uc REQUIRED)
endif()

if(WITH_OPENSSL)
    set(OPENSSL_ROOT_DIR $ENV{ASV_PLAT_PORTS})
    find_package(OpenSSL REQUIRED)
endif()

if(WIN32)
    option(ONECORE "Use OneCore libraries" OFF)
endif()

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

CHECK_INCLUDE_FILE_CXX("format" HAS_STD_FORMAT)

if(NOT HAS_STD_FORMAT)
    find_package(fmt REQUIRED)
endif()

# ----------------------------------------

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# -----------------------
# Public header list
# -----------------------
set(public_headers
    src/tdscpp.h
)

# Define library
add_library(${libname} SHARED)
asv_optimize_target(${libname})

# Use file sets for public headers
target_sources(${libname} PUBLIC
    FILE_SET HEADERS
    BASE_DIRS src
    FILES ${public_headers}
)

# Source files
target_sources(${libname} PRIVATE
    src/tdscpp.cpp
    src/bcp.cpp
    src/value.cpp
    src/ssl.cpp
    src/ringbuf.cpp
    src/rpc.cpp
    src/batch.cpp
    src/collation.cpp
    src/ver80coll.cpp
    src/ver90coll.cpp
    src/ver100coll.cpp
    src/ver140coll.cpp
    src/tests.cpp
)

# Include paths
target_include_directories(${libname}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

if(WIN32)
    add_definitions(-DNOMINMAX)
endif()

target_link_libraries(${libname} PUBLIC nlohmann_json::nlohmann_json)

if(WITH_OPENSSL)
    target_link_libraries(${libname} PRIVATE OpenSSL::Crypto OpenSSL::SSL)
endif()

if(NOT WIN32)
    target_link_libraries(${libname} PRIVATE ICU::uc)
endif()

if(NOT HAS_STD_FORMAT)
    target_link_libraries(${libname} PRIVATE fmt::fmt-header-only)
endif()

target_compile_options(${libname} PRIVATE
     $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:GNU>>:
          -Wall>
     $<$<CXX_COMPILER_ID:MSVC>:
          /W4>)

if(NOT MSVC)
    target_compile_options(${libname} PUBLIC -Wall -Werror=cast-function-type -Wno-expansion-to-defined -Wunused-parameter -Wtype-limits -Wextra -Wconversion -fstrict-aliasing)
endif()

if(WIN32)
    if(ONECORE)
        target_link_options(${libname} PUBLIC -nodefaultlibs)
        target_link_libraries(${libname} PRIVATE -Wl,-Bstatic -lstdc++ -Wl,-Bdynamic)
        target_link_libraries(${libname} PRIVATE gcc mingwex mingw32 gcc_eh ucrt onecore_apiset)
    else()
        target_link_libraries(${libname} PRIVATE ws2_32 secur32 crypt32)
    endif()
elseif(ENABLE_KRB5)
    target_link_libraries(${libname} PRIVATE PkgConfig::KRB5)
endif()

# Install library and headers
install(TARGETS ${libname}
    EXPORT ${PROJECT_NAME}Targets
    FILE_SET HEADERS DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

# Export targets
install(EXPORT ${PROJECT_NAME}Targets
    FILE ${PROJECT_NAME}Targets.cmake
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION lib/cmake/${PROJECT_NAME}
)

if(MSVC)
    install(FILES $<TARGET_PDB_FILE:${libname}> DESTINATION bin OPTIONAL)
endif()

# Generate and install CMake config files
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION lib/cmake/${PROJECT_NAME}
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    DESTINATION lib/cmake/${PROJECT_NAME}
)

# ----------------------------------------

if(BUILD_SAMPLE)
    add_executable(tdscpp-test src/tdscpp-test.cpp)
    target_link_libraries(tdscpp-test ${libname})

    if(NOT HAS_STD_FORMAT)
        target_link_libraries(tdscpp-test fmt::fmt-header-only)
    endif()

    install(TARGETS tdscpp-test DESTINATION ${CMAKE_INSTALL_BINDIR})

    add_executable(colltest src/colltest.cpp)
    target_link_libraries(colltest ${libname})

    if(NOT HAS_STD_FORMAT)
        target_link_libraries(colltest fmt::fmt-header-only)
    endif()

    install(TARGETS colltest DESTINATION ${CMAKE_INSTALL_BINDIR})

    if(MSVC)
        install(FILES $<TARGET_PDB_FILE:tdscpp-test> DESTINATION bin OPTIONAL)
        install(FILES $<TARGET_PDB_FILE:colltest> DESTINATION bin OPTIONAL)
    endif()
endif()

# CPack packaging
set(CPACK_PACKAGE_NAME tdscpp-dev)
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Mark Harmstone")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libicu-dev, nlohmann-json3-dev, libssl-dev, libkrb5-dev")
include(CPack)
