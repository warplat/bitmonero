# Copyright (c) 2014-2016, The Monero Project
#
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without modification, are
# permitted provided that the following conditions are met:
#
# 1. Redistributions of source code must retain the above copyright notice, this list of
#    conditions and the following disclaimer.
#
# 2. Redistributions in binary form must reproduce the above copyright notice, this list
#    of conditions and the following disclaimer in the documentation and/or other
#    materials provided with the distribution.
#
# 3. Neither the name of the copyright holder nor the names of its contributors may be
#    used to endorse or promote products derived from this software without specific
#    prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
# MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL
# THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
# INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
# STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF
# THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#
# Parts of this file are originally copyright (c) 2012-2013 The Cryptonote developers

cmake_minimum_required(VERSION 2.8.7)

project(bitmonero)

function (die msg)
  if (NOT WIN32)
    string(ASCII 27 Esc)
    set(ColourReset "${Esc}[m")
    set(BoldRed     "${Esc}[1;31m")
  else ()
    set(ColourReset "")
    set(BoldRed     "")
  endif ()

  message(FATAL_ERROR "${BoldRed}${msg}${ColourReset}")
endfunction ()

if ("${ARCH}" STREQUAL "" OR "${ARCH}" STREQUAL "native")
  set(ARCH ${CMAKE_SYSTEM_PROCESSOR})
  message(STATUS "Building natively on ${ARCH}")
endif()

if (NOT "${ARCH}" STREQUAL "")
  string(SUBSTRING ${ARCH} 0 3 IS_ARM)
  string(TOLOWER ${IS_ARM} IS_ARM)

  if (${IS_ARM} STREQUAL "arm")
    string(SUBSTRING ${ARCH} 0 5 ARM_TEST)
    string(TOLOWER ${ARM_TEST} ARM_TEST)

    if (${ARM_TEST} STREQUAL "armv6")
      set(ARM6 1)
    else()
      set(ARM6 0)
    endif()

    if (${ARM_TEST} STREQUAL "armv7")
      set(ARM7 1)
    else()
      set(ARM7 0)
    endif()
  endif()
endif()

if(WIN32 OR ARM7 OR ARM6)
  set(OPT_FLAGS_RELEASE "-O2")
else()
  set(OPT_FLAGS_RELEASE "-Ofast")
endif()

set(CMAKE_C_FLAGS_RELEASE "-DNDEBUG ${OPT_FLAGS_RELEASE}")
set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG ${OPT_FLAGS_RELEASE}")

# set this to 0 if per-block checkpoint needs to be disabled
set(PER_BLOCK_CHECKPOINT 1)

if(PER_BLOCK_CHECKPOINT)
  add_definitions("-DPER_BLOCK_CHECKPOINT")
endif()

list(INSERT CMAKE_MODULE_PATH 0
  "${CMAKE_SOURCE_DIR}/cmake")

if (NOT DEFINED ENV{DEVELOPER_LOCAL_TOOLS})
  message(STATUS "Could not find DEVELOPER_LOCAL_TOOLS in env (not required)")
  set(BOOST_IGNORE_SYSTEM_PATHS_DEFAULT OFF)
elseif ("$ENV{DEVELOPER_LOCAL_TOOLS}" EQUAL 1)
  message(STATUS "Found: env DEVELOPER_LOCAL_TOOLS = 1")
  set(BOOST_IGNORE_SYSTEM_PATHS_DEFAULT ON)
else()
  message(STATUS "Found: env DEVELOPER_LOCAL_TOOLS = 0")
  set(BOOST_IGNORE_SYSTEM_PATHS_DEFAULT OFF)
endif()

message(STATUS "BOOST_IGNORE_SYSTEM_PATHS defaults to ${BOOST_IGNORE_SYSTEM_PATHS_DEFAULT}")
option(BOOST_IGNORE_SYSTEM_PATHS "Ignore boost system paths for local boost installation" ${BOOST_IGNORE_SYSTEM_PATHS_DEFAULT})


if (NOT DEFINED ENV{DEVELOPER_LIBUNBOUND_OLD})
  message(STATUS "Could not find DEVELOPER_LIBUNBOUND_OLD in env (not required)")
elseif ("$ENV{DEVELOPER_LIBUNBOUND_OLD}" EQUAL 1)
  message(STATUS "Found: env DEVELOPER_LIBUNBOUND_OLD = 1, will use the work around")
  add_definitions(-DDEVELOPER_LIBUNBOUND_OLD)
elseif ("$ENV{DEVELOPER_LIBUNBOUND_OLD}" EQUAL 0)
  message(STATUS "Found: env DEVELOPER_LIBUNBOUND_OLD = 0")
else()
  message(STATUS "Found: env DEVELOPER_LIBUNBOUND_OLD with bad value. Will NOT use the work around")
endif()

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
enable_testing()

option(BUILD_DOCUMENTATION "Build the Doxygen documentation." ON)

# Check whether we're on a 32-bit or 64-bit system
if(CMAKE_SIZEOF_VOID_P EQUAL "8")
  set(DEFAULT_BUILD_64 ON)
else()
  set(DEFAULT_BUILD_64 OFF)
endif()
option(BUILD_64 "Build for 64-bit? 'OFF' builds for 32-bit." ${DEFAULT_BUILD_64})

if(BUILD_64)
  set(ARCH_WIDTH "64")
else()
  set(ARCH_WIDTH "32")
endif()
message(STATUS "Building for a ${ARCH_WIDTH}-bit system")

# Check if we're on FreeBSD so we can exclude the local miniupnpc (it should be installed from ports instead)
# CMAKE_SYSTEM_NAME checks are commonly known, but specifically taken from libsdl's CMakeLists
if(CMAKE_SYSTEM_NAME MATCHES "kFreeBSD.*")
  set(FREEBSD TRUE)
elseif(CMAKE_SYSTEM_NAME MATCHES "DragonFly.*|FreeBSD")
  set(FREEBSD TRUE)
endif()

# Check if we're on OpenBSD. See the README.md for build instructions.
if(CMAKE_SYSTEM_NAME MATCHES "kOpenBSD.*|OpenBSD.*")
  set(OPENBSD TRUE)
endif()

# TODO: check bsdi, NetBSD, to see if they need the same FreeBSD changes
#
# elseif(CMAKE_SYSTEM_NAME MATCHES "kNetBSD.*|NetBSD.*")
#   set(NETBSD TRUE)
# elseif(CMAKE_SYSTEM_NAME MATCHES ".*BSDI.*")
#   set(BSDI TRUE)

include_directories(src contrib/epee/include external "${CMAKE_BINARY_DIR}/version")

if(APPLE)
  include_directories(SYSTEM /usr/include/malloc)
endif()

if(MSVC OR MINGW)
  set(DEFAULT_STATIC true)
else()
  set(DEFAULT_STATIC false)
endif()
option(STATIC "Link libraries statically" ${DEFAULT_STATIC})

if(MINGW)
  string(REGEX MATCH "^[^/]:/[^/]*" msys2_install_path "${CMAKE_C_COMPILER}")
  message(STATUS "MSYS location: ${msys2_install_path}")
  set(CMAKE_INCLUDE_PATH "${msys2_install_path}/mingw${ARCH_WIDTH}/include")
  # This is necessary because otherwise CMake will make Boost libraries -lfoo
  # rather than a full path. Unfortunately, this makes the shared libraries get
  # linked due to a bug in CMake which misses putting -static flags around the
  # -lfoo arguments.
  set(DEFLIB ${msys2_install_path}/mingw${ARCH_WIDTH}/lib)
  list(REMOVE_ITEM CMAKE_C_IMPLICIT_LINK_DIRECTORIES ${DEFLIB})
  list(REMOVE_ITEM CMAKE_CXX_IMPLICIT_LINK_DIRECTORIES ${DEFLIB})
endif()

if(STATIC)
  if(MSVC)
    set(CMAKE_FIND_LIBRARY_SUFFIXES .lib .dll.a .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
  else()
    set(CMAKE_FIND_LIBRARY_SUFFIXES .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
  endif()
endif()

# default database:
# should be lmdb for testing, memory for production still
# set(DATABASE memory)
set(DATABASE lmdb)

if (DEFINED ENV{DATABASE})
  set(DATABASE $ENV{DATABASE})
  message(STATUS "DATABASE set: ${DATABASE}")
else()
  message(STATUS "Could not find DATABASE in env (not required unless you want to change database type from default: ${DATABASE})")
endif()

set(BERKELEY_DB_OVERRIDE 0)
if (DEFINED ENV{BERKELEY_DB})
  set(BERKELEY_DB_OVERRIDE 1)
  set(BERKELEY_DB $ENV{BERKELEY_DB})
elseif()
  set(BERKELEY_DB 0)
endif()

if (DATABASE STREQUAL "lmdb")
  message(STATUS "Using LMDB as default DB type")
  set(BLOCKCHAIN_DB DB_LMDB)
  add_definitions("-DDEFAULT_DB_TYPE=\"lmdb\"")
elseif (DATABASE STREQUAL "berkeleydb")
  find_package(BerkeleyDB)
  if(NOT BERKELEY_DB)
      die("Found BerkeleyDB includes, but could not find BerkeleyDB library. Please make sure you have installed libdb and libdb-dev / libdb++-dev or the equivalent.")
  else()
    message(STATUS "Found BerkeleyDB include (db.h) in ${BERKELEY_DB_INCLUDE_DIR}")
    if(BERKELEY_DB_LIBRARIES)
      message(STATUS "Found BerkeleyDB shared library")
      set(BDB_STATIC false CACHE BOOL "BDB Static flag")
      set(BDB_INCLUDE ${BERKELEY_DB_INCLUDE_DIR} CACHE STRING "BDB include path")
      set(BDB_LIBRARY ${BERKELEY_DB_LIBRARIES} CACHE STRING "BDB library name")
      set(BDB_LIBRARY_DIRS "" CACHE STRING "BDB Library dirs")
      set(BERKELEY_DB 1)
    else()
      die("Found BerkeleyDB includes, but could not find BerkeleyDB library. Please make sure you have installed libdb and libdb-dev / libdb++-dev or the equivalent.")
    endif()
  endif()

  message(STATUS "Using Berkeley DB as default DB type")
  add_definitions("-DDEFAULT_DB_TYPE=\"berkeley\"")
elseif (DATABASE STREQUAL "memory")
  set(BLOCKCHAIN_DB DB_MEMORY)
  message(STATUS "Using Serialised In Memory as default DB type")
  add_definitions("-DDEFAULT_DB_TYPE=\"memory\"")
else()
  die("Invalid database type: ${DATABASE}")
endif()

if(BERKELEY_DB)
  add_definitions("-DBERKELEY_DB")
endif()

add_definitions("-DBLOCKCHAIN_DB=${BLOCKCHAIN_DB}")

find_package(Libunwind)
# Can't install hook in static build on OSX, because OSX linker does not support --wrap
if(LIBUNWIND_FOUND AND NOT (STATIC AND APPLE))
  set(DEFAULT_STACK_TRACE ON)
else()
  set(DEFAULT_STACK_TRACE OFF)
  set(LIBUNWIND_LIBRARIES "")
endif()

option(STACK_TRACE "Install a hook that dumps stack on exception" ${DEFAULT_STACK_TRACE})

if(STACK_TRACE)
  message(STATUS "Stack trace on exception enabled")
else()
  message(STATUS "Stack trace on exception disabled")
endif()

if (UNIX AND NOT APPLE)
  # Note that at the time of this writing the -Wstrict-prototypes flag added below will make this fail
  set(THREADS_PREFER_PTHREAD_FLAG ON)
  find_package(Threads)
endif()

add_subdirectory(external)

# Final setup for miniupnpc
if(UPNP_STATIC)
  add_definitions("-DUPNP_STATIC")
else()
  add_definitions("-DUPNP_DYNAMIC")
  include_directories(${UPNP_INCLUDE})
endif()

# Final setup for libunbound
include_directories(${UNBOUND_INCLUDE})
link_directories(${UNBOUND_LIBRARY_DIRS})

# Final setup for rapidjson
include_directories(external/rapidjson)

# Final setup for liblmdb
include_directories(${LMDB_INCLUDE})

# Final setup for Berkeley DB
if (BERKELEY_DB)
  include_directories(${BDB_INCLUDE})
endif()

# Final setup for libunwind
include_directories(${LIBUNWIND_INCLUDE})
link_directories(${LIBUNWIND_LIBRARY_DIRS})

if(MSVC)
  add_definitions("/bigobj /MP /W3 /GS- /D_CRT_SECURE_NO_WARNINGS /wd4996 /wd4345 /D_WIN32_WINNT=0x0600 /DWIN32_LEAN_AND_MEAN /DGTEST_HAS_TR1_TUPLE=0 /FIinline_c.h /D__SSE4_1__")
  # set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /Dinline=__inline")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /STACK:10485760")
  if(STATIC)
    foreach(VAR CMAKE_C_FLAGS_DEBUG CMAKE_CXX_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE CMAKE_CXX_FLAGS_RELEASE)
      string(REPLACE "/MD" "/MT" ${VAR} "${${VAR}}")
    endforeach()
  endif()
  include_directories(SYSTEM src/platform/msc)
else()
  set(ARCH native CACHE STRING "CPU to build for: -march value or default")
  # -march=armv7-a conflicts with -mcpu=cortex-a7
  if(ARCH STREQUAL "default" OR ARM7 OR ARM6)
    set(ARCH_FLAG "")
  else()
    if(ARCH STREQUAL "x86_64")
      set(ARCH_FLAG "-march=x86-64")
    else()
       set(ARCH_FLAG "-march=${ARCH}")
    endif()
  endif()
  set(WARNINGS "-Wall -Wextra -Wpointer-arith -Wundef -Wvla -Wwrite-strings -Wno-error=extra -Wno-error=deprecated-declarations -Wno-unused-parameter -Wno-unused-variable -Wno-error=unused-variable -Wno-error=undef -Wno-error=uninitialized")
  if(NOT MINGW)
    set(WARNINGS_AS_ERRORS_FLAG "-Werror")
  endif()
  if(CMAKE_C_COMPILER_ID STREQUAL "Clang")
    set(WARNINGS "${WARNINGS} -Wno-deprecated-register -Wno-error=mismatched-tags -Wno-error=null-conversion -Wno-overloaded-shift-op-parentheses -Wno-error=shift-count-overflow -Wno-error=tautological-constant-out-of-range-compare -Wno-error=unused-private-field -Wno-error=unneeded-internal-declaration")
    if(ARM6 OR ARM7)
      set(WARNINGS "${WARNINGS} -Wno-error=inline-asm")
    endif()
  else()
    set(WARNINGS "${WARNINGS} -Wlogical-op -Wno-error=maybe-uninitialized")
  endif()
  if(MINGW)
    set(WARNINGS "${WARNINGS} -Wno-error=unused-value -Wno-error=unused-but-set-variable")
    set(MINGW_FLAG "${MINGW_FLAG} -DWIN32_LEAN_AND_MEAN")
    set(Boost_THREADAPI win32)
    include_directories(SYSTEM src/platform/mingw)
    # mingw doesn't support LTO (multiple definition errors at link time)
    set(USE_LTO_DEFAULT false)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--stack,10485760")
    if(NOT BUILD_64)
      add_definitions(-DWINVER=0x0501 -D_WIN32_WINNT=0x0501)
    endif()
  endif()
  set(C_WARNINGS "-Waggregate-return -Wnested-externs -Wold-style-definition -Wstrict-prototypes")
  set(CXX_WARNINGS "-Wno-reorder -Wno-missing-field-initializers")
  try_compile(STATIC_ASSERT_RES "${CMAKE_CURRENT_BINARY_DIR}/static-assert" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/test-static-assert.c" COMPILE_DEFINITIONS "-std=c11")
  if(STATIC_ASSERT_RES)
    set(STATIC_ASSERT_FLAG "")
  else()
    set(STATIC_ASSERT_FLAG "-Dstatic_assert=_Static_assert")
  endif()

  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 -D_GNU_SOURCE ${MINGW_FLAG} ${STATIC_ASSERT_FLAG} ${WARNINGS} ${C_WARNINGS} ${ARCH_FLAG}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -D_GNU_SOURCE ${MINGW_FLAG} ${WARNINGS} ${CXX_WARNINGS} ${ARCH_FLAG}")

  # With GCC 6.1.1 the compiled binary malfunctions due to aliasing. Until that
  # is fixed in the code (Issue #847), force compiler to be conservative.
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-strict-aliasing")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-strict-aliasing")

  option(NO_AES "Explicitly disable AES support" ${NO_AES})

  if(NOT NO_AES AND NOT (ARM6 OR ARM7))
    message(STATUS "AES support enabled")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -maes")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maes")
  elseif(ARM7 OR ARM6)
    message(STATUS "AES support disabled (not available on ARM)")
  else()
    message(STATUS "AES support disabled")
  endif()

  if(ARM6)
    message(STATUS "Setting ARM6 C and C++ flags")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfpu=vfp -mfloat-abi=hard")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfpu=vfp -mfloat-abi=hard")
  endif()

  if(ARM7)
    message(STATUS "Setting ARM7 C and C++ flags")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfloat-abi=hard")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfloat-abi=hard")
  endif()

  if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGTEST_HAS_TR1_TUPLE=0")
  endif()
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_C_COMPILER_VERSION VERSION_LESS 4.8))
    set(DEBUG_FLAGS "-g3 -Og")
  else()
    set(DEBUG_FLAGS "-g3 -O0")
  endif()

  if(NOT DEFINED USE_LTO_DEFAULT)
    set(USE_LTO_DEFAULT true)
  endif()
  set(USE_LTO ${USE_LTO_DEFAULT} CACHE BOOL "Use Link-Time Optimization (Release mode only)")

  if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    # There is a clang bug that does not allow to compile code that uses AES-NI intrinsics if -flto is enabled, so explicitly disable
    set(USE_LTO false)
    # explicitly define stdlib for older versions of clang
    if(CMAKE_C_COMPILER_VERSION VERSION_LESS 3.7)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libstdc++")
      set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -stdlib=libstdc++")
    endif()
  endif()


  if(USE_LTO)
    set(RELEASE_FLAGS "${RELEASE_FLAGS} -flto")
    if(STATIC)
      set(RELEASE_FLAGS "${RELEASE_FLAGS} -ffat-lto-objects")
    endif()
    # Since gcc 4.9 the LTO format is non-standard (slim), so we need the gcc-specific ar and ranlib binaries
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND NOT (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9.0) AND NOT OPENBSD)
      set(CMAKE_AR "gcc-ar")
      set(CMAKE_RANLIB "gcc-ranlib")
    endif()
  endif()

  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${DEBUG_FLAGS}")
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${RELEASE_FLAGS}")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${RELEASE_FLAGS}")

  if(STATIC AND NOT APPLE AND NOT FREEBSD AND NOT OPENBSD)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++")
  endif()
endif()

if (${BOOST_IGNORE_SYSTEM_PATHS} STREQUAL "ON")
  set(Boost_NO_SYSTEM_PATHS TRUE)
endif()

set(OLD_LIB_SUFFIXES ${CMAKE_FIND_LIBRARY_SUFFIXES})
if(STATIC)
  if(MINGW)
    set(CMAKE_FIND_LIBRARY_SUFFIXES .a)
  endif()

  set(Boost_USE_STATIC_LIBS ON)
  set(Boost_USE_STATIC_RUNTIME ON)
endif()
find_package(Boost 1.53 QUIET REQUIRED COMPONENTS system filesystem thread date_time chrono regex serialization program_options)

set(CMAKE_FIND_LIBRARY_SUFFIXES ${OLD_LIB_SUFFIXES})
if(NOT Boost_FOUND)
  die("Could not find Boost libraries, please make sure you have installed Boost or libboost-all-dev (1.53 or 1.55+) or the equivalent")
endif()

if((Boost_MAJOR_VERSION EQUAL 1) AND (Boost_MINOR_VERSION EQUAL 54))
  die("Boost version 1.54 is unsupported due to a bug (see: http://goo.gl/RrCFmA), please install Boost 1.53 or 1.55 and above")
endif()

include_directories(SYSTEM ${Boost_INCLUDE_DIRS})
if(MINGW)
  set(EXTRA_LIBRARIES mswsock;ws2_32;iphlpapi)
elseif(APPLE OR FREEBSD OR OPENBSD)
  set(EXTRA_LIBRARIES "")
elseif(NOT MSVC)
  find_library(RT rt)
  set(EXTRA_LIBRARIES ${RT})
endif()

list(APPEND EXTRA_LIBRARIES ${CMAKE_DL_LIBS})

if(CMAKE_C_COMPILER_ID STREQUAL "Clang" AND ARCH_WIDTH EQUAL "32")
  find_library(ATOMIC atomic)
  list(APPEND EXTRA_LIBRARIES ${ATOMIC})
endif()

include(version.cmake)

function (treat_warnings_as_errors dirs)
  foreach(dir ${ARGV})
    set_property(DIRECTORY ${dir}
      APPEND PROPERTY COMPILE_FLAGS "-Werror")
  endforeach()
endfunction()

add_subdirectory(contrib)
add_subdirectory(src)

treat_warnings_as_errors(contrib src)

option(BUILD_TESTS "Build tests." OFF)

if(BUILD_TESTS)
  add_subdirectory(tests)
endif()



if(BUILD_DOCUMENTATION)
  set(DOC_GRAPHS "YES" CACHE STRING "Create dependency graphs (needs graphviz)")
  set(DOC_FULLGRAPHS "NO" CACHE STRING "Create call/callee graphs (large)")

  find_program(DOT_PATH dot)

  if (DOT_PATH STREQUAL "DOT_PATH-NOTFOUND")
    message("Doxygen: graphviz not found - graphs disabled")
    set(DOC_GRAPHS "NO")
  endif()

  find_package(Doxygen)
  if(DOXYGEN_FOUND)
    configure_file("cmake/Doxyfile.in" "Doxyfile" @ONLY)
    configure_file("cmake/Doxygen.extra.css.in" "Doxygen.extra.css" @ONLY)
    add_custom_target(doc
      ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
      COMMENT "Generating API documentation with Doxygen.." VERBATIM)
  endif()
endif()

