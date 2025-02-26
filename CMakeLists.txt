# SPDX-FileCopyrightText: 2011-2023 Greenbone AG
#
# SPDX-License-Identifier: GPL-2.0-or-later

cmake_minimum_required(VERSION 3.0)

message ("-- Configuring the Greenbone Vulnerability Management Libraries...")

# VERSION: Always include major, minor and patch level.
project (gvm-libs
  VERSION 22.9.1
  LANGUAGES C)

if (POLICY CMP0005)
  cmake_policy (SET CMP0005 NEW)
endif (POLICY CMP0005)

set (C_STANDARD, 11)
set (CMAKE_C_STANDARD 11)
if (NOT CMAKE_BUILD_TYPE)
  set (CMAKE_BUILD_TYPE Debug)
endif (NOT CMAKE_BUILD_TYPE)

OPTION(BUILD_STATIC "Build static versions of the libraries" OFF)
OPTION(ENABLE_COVERAGE "Enable support for coverage analysis" OFF)
OPTION(BUILD_TESTS "Build tests for the libraries" OFF)

if (NOT BUILD_STATIC)
  set (BUILD_SHARED ON)
else (NOT BUILD_STATIC)
  set (BUILD_SHARED OFF)
endif (NOT BUILD_STATIC)

## Retrieve git revision (at configure time)
find_package (Git)

## make format
message (STATUS "Looking for clang-format...")
find_program (CLANG_FORMAT clang-format)

if (CLANG_FORMAT)
  message (STATUS "Looking for clang-format... ${CLANG_FORMAT}")
  add_custom_target(format COMMAND ${CLANG_FORMAT} "-i" "./base/*.c" 
                    "./boreas/*.c" "./gmp/*.c" "./osp/*.c" "./util/*.c" 
                    "./base/*.h" "./boreas/*.h" "./gmp/*.h" "./osp/*.h"
                    "./util/*.h" WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}")
else (CLANG_FORMAT)
  message (STATUS "clang-format not found...")
endif (CLANG_FORMAT)


macro (Git_GET_REVISION dir variable)
  execute_process(COMMAND ${GIT_EXECUTABLE} rev-parse --abbrev-ref HEAD
                  WORKING_DIRECTORY ${dir}
                  OUTPUT_VARIABLE GIT_BRANCH
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  execute_process(COMMAND ${GIT_EXECUTABLE} log -1 --format=%h
                  WORKING_DIRECTORY ${dir}
                  OUTPUT_VARIABLE GIT_COMMIT_HASH
                  OUTPUT_STRIP_TRAILING_WHITESPACE)
  string (REPLACE "/" "_" GIT_BRANCH ${GIT_BRANCH})
  set (${variable} "${GIT_COMMIT_HASH}-${GIT_BRANCH}")
endmacro (Git_GET_REVISION)

if (NOT CMAKE_BUILD_TYPE MATCHES "Release")
  if (EXISTS "${CMAKE_SOURCE_DIR}/.git/")
    if (GIT_FOUND)
      Git_GET_REVISION(${CMAKE_SOURCE_DIR} ProjectRevision)
      set (GIT_REVISION "~git-${ProjectRevision}")
    else (GIT_FOUND)
      set (GIT_REVISION "~git")
    endif (GIT_FOUND)
  endif (EXISTS "${CMAKE_SOURCE_DIR}/.git/")
endif (NOT CMAKE_BUILD_TYPE MATCHES "Release")

# Set dev version if this is a development version and not a full release,
# unset (put value 0 or delete line) before a full release and reset after.
set (PROJECT_DEV_VERSION 0)

# If PROJECT_DEV_VERSION is set, the version string will be set to:
#   "major.minor.patch~dev${PROJECT_DEV_VERSION}${GIT_REVISION}"
# If PROJECT_DEV_VERSION is NOT set, the version string will be set to:
#   "major.minor.patch${GIT_REVISION}"
# For CMAKE_BUILD_TYPE "Release" the git revision will be empty.
if (PROJECT_DEV_VERSION)
  set (PROJECT_VERSION_SUFFIX "~dev${PROJECT_DEV_VERSION}")
endif (PROJECT_DEV_VERSION)

set (PROJECT_VERSION_STRING "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}${PROJECT_VERSION_SUFFIX}${GIT_REVISION}")

## CPack configuration

set (CPACK_CMAKE_GENERATOR "Unix Makefiles")
set (CPACK_GENERATOR "TGZ")
set (CPACK_INSTALL_CMAKE_PROJECTS ".;gvm-libs;ALL;/")
set (CPACK_MODULE_PATH "")
set (CPACK_RESOURCE_FILE_LICENSE "${CMAKE_SOURCE_DIR}/COPYING")
set (CPACK_RESOURCE_FILE_README "${CMAKE_SOURCE_DIR}/README.md")
set (CPACK_RESOURCE_FILE_WELCOME "${CMAKE_SOURCE_DIR}/README.md")
set (CPACK_SOURCE_GENERATOR "TGZ")
set (CPACK_SOURCE_TOPLEVEL_TAG "")
set (CPACK_SYSTEM_NAME "")
set (CPACK_TOPLEVEL_TAG "")

set (CPACK_PACKAGE_VERSION "${PROJECT_VERSION_STRING}${PROJECT_VERSION_GIT}")

set (CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}-${CPACK_PACKAGE_VERSION}")
set (CPACK_SOURCE_PACKAGE_FILE_NAME "${PROJECT_NAME}-${CPACK_PACKAGE_VERSION}")
set (CPACK_PACKAGE_VENDOR "Greenbone Networks")
set (CPACK_SOURCE_IGNORE_FILES
"${CMAKE_BINARY_DIR}"
"/.git/"
"swp$"
)

include (CPack)

## Variables

if (SYSCONF_INSTALL_DIR)
  set (SYSCONFDIR "${SYSCONF_INSTALL_DIR}")
endif (SYSCONF_INSTALL_DIR)

if (NOT SYSCONFDIR)
  set (SYSCONFDIR "/etc")
endif (NOT SYSCONFDIR)

if (NOT EXEC_PREFIX)
  set (EXEC_PREFIX "${CMAKE_INSTALL_PREFIX}")
endif (NOT EXEC_PREFIX)

if (NOT BINDIR)
  set (BINDIR "${EXEC_PREFIX}/bin")
endif (NOT BINDIR)

if (NOT SBINDIR)
  set (SBINDIR "${EXEC_PREFIX}/sbin")
endif (NOT SBINDIR)

if (NOT LIBDIR)
  set (_DEFAULT_LIBRARY_INSTALL_DIR lib)
  if (EXISTS "${EXEC_PREFIX}/lib32/" AND CMAKE_SIZEOF_VOID_P EQUAL 4)
    set (_DEFAULT_LIBRARY_INSTALL_DIR lib32)
  endif (EXISTS "${EXEC_PREFIX}/lib32/" AND CMAKE_SIZEOF_VOID_P EQUAL 4)
  if (EXISTS "${CMAKE_INSTALL_PREFIX}/lib64/" AND CMAKE_SIZEOF_VOID_P EQUAL 8)
    set (_DEFAULT_LIBRARY_INSTALL_DIR lib64)
  endif (EXISTS "${CMAKE_INSTALL_PREFIX}/lib64/" AND CMAKE_SIZEOF_VOID_P EQUAL 8)
  set( LIBRARY_INSTALL_DIR "${_DEFAULT_LIBRARY_INSTALL_DIR}")
  set (LIBDIR "${EXEC_PREFIX}/${LIBRARY_INSTALL_DIR}")
endif (NOT LIBDIR)

if (NOT LOCALSTATEDIR)
  set (LOCALSTATEDIR "/var")
endif (NOT LOCALSTATEDIR)

if (NOT INCLUDEDIR)
  set (INCLUDEDIR "${CMAKE_INSTALL_PREFIX}/include")
endif (NOT INCLUDEDIR)

if (NOT DATADIR)
  set (DATADIR "${CMAKE_INSTALL_PREFIX}/share")
endif (NOT DATADIR)

if (NOT GVM_RUN_DIR)
  set (GVM_RUN_DIR  "/run/gvm")
endif (NOT GVM_RUN_DIR)

if (NOT GVM_SYSCONF_DIR)
  set (GVM_SYSCONF_DIR "${SYSCONFDIR}/gvm")
endif (NOT GVM_SYSCONF_DIR)


if (NOT REDIS_SOCKET_PATH)
  set (REDIS_SOCKET_PATH "/run/redis/redis.sock")
endif ()

add_definitions (-DREDIS_SOCKET_PATH="${REDIS_SOCKET_PATH}")
message ("-- Using redis socket ${REDIS_SOCKET_PATH}")

message ("-- Install prefix: ${CMAKE_INSTALL_PREFIX}")

if (ENABLE_COVERAGE)
  set (COVERAGE_FLAGS "--coverage")
endif (ENABLE_COVERAGE)

set (HARDENING_FLAGS "-Wformat -Wformat-security -D_FORTIFY_SOURCE=2 -fstack-protector")
set (LINKER_HARDENING_FLAGS "-Wl,-z,relro -Wl,-z,now")
set (CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${COVERAGE_FLAGS}")
set (CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${HARDENING_FLAGS}")

set (
    CMAKE_C_FLAGS
    "${CMAKE_C_FLAGS} ${GPGME_C_FLAGS} \
    -std=c11 \
    -Wall \
    -Wextra \
    -Werror \
    -Wpedantic \
    -Wmissing-prototypes \
    -Wshadow \
    -Wsequence-point \
    -D_ISOC11_SOURCE \
    -D_DEFAULT_SOURCE"
)

## Version


set (LIBGVMCONFIG_VERSION "${PROJECT_VERSION_STRING}")

add_definitions(-DGVM_LIBS_VERSION="${LIBGVMCONFIG_VERSION}")

# Configure Doxyfile with version number
configure_file (doc/Doxyfile.in ${CMAKE_BINARY_DIR}/doc/Doxyfile @ONLY)
configure_file (doc/Doxyfile_full.in ${CMAKE_BINARY_DIR}/doc/Doxyfile_full @ONLY)
configure_file (doc/Doxyfile_xml.in ${CMAKE_BINARY_DIR}/doc/Doxyfile_xml @ONLY)
configure_file (VERSION.in ${CMAKE_BINARY_DIR}/VERSION @ONLY)

## Testing

if (BUILD_TESTS AND NOT SKIP_SRC)
  message ("-- Tests enabled.")
  enable_testing ()

  find_package (cgreen REQUIRED)

  add_custom_target (tests
    DEPENDS array-test alivedetection-test boreas_error-test boreas_io-test
            cli-test cvss-test ping-test sniffer-test util-test networking-test
            passwordbasedauthentication-test xmlutils-test version-test osp-test 
            nvti-test hosts-test)

endif (BUILD_TESTS AND NOT SKIP_SRC)

## Program

if (NOT SKIP_SRC)
  add_subdirectory (base)
  add_subdirectory (boreas)
  add_subdirectory (util)
  add_subdirectory (osp)
  add_subdirectory (gmp)
endif (NOT SKIP_SRC)

## Documentation

add_subdirectory (doc)

if (BUILD_TESTS AND NOT SKIP_SRC)
  add_subdirectory (tests)
  add_test (NAME testhosts COMMAND test-hosts localhost)
endif (BUILD_TESTS AND NOT SKIP_SRC)

## End
