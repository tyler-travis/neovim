cmake_minimum_required (VERSION 2.8.7)
project (NEOVIM)

set(CMAKE_SYSTEM_PROCESSOR i386)
set(CMAKE_SYSTEM_LIBRARY_PATH /lib32 /usr/lib32 /usr/local/lib32)
set(FIND_LIBRARY_USE_LIB64_PATHS OFF)
set(CMAKE_IGNORE_PATH /lib /usr/lib /usr/local/lib)

# Point CMake at any custom modules we may ship
set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

# Prefer our bundled versions of dependencies.
set(DEPS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/.deps")
set(DEPS_BUILD_DIR "${DEPS_DIR}/build")
set(DEPS_INSTALL_DIR "${DEPS_DIR}/usr")
set(DEPS_BIN_DIR "${DEPS_INSTALL_DIR}/bin")

list(APPEND CMAKE_PREFIX_PATH ${DEPS_INSTALL_DIR})

set(NEOVIM_VERSION_MAJOR 0)
set(NEOVIM_VERSION_MINOR 0)
set(NEOVIM_VERSION_PATCH 0)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

add_definitions(-Werror -Wall -Wextra -pedantic -Wno-unused-parameter -std=gnu99)

add_definitions(-DHAVE_CONFIG_H)
if(CMAKE_BUILD_TYPE MATCHES Debug)
  # cmake automatically appends -g to the compiler flags
  set(DEBUG 1)
else()
  set(DEBUG 0)
endif()

option(USE_GCOV "Enable gcov support" OFF)

if(USE_GCOV)
  message(STATUS "Enabling gcov support")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} --coverage")
endif()

include_directories("${PROJECT_BINARY_DIR}/config")
include_directories("${PROJECT_SOURCE_DIR}/src")

# Modules used by platform auto-detection
include(CheckLibraryExists)

find_package(LibUV REQUIRED)
include_directories(${LIBUV_INCLUDE_DIRS})

find_package(Msgpack REQUIRED)
include_directories(${MSGPACK_INCLUDE_DIRS})

find_package(LuaJit REQUIRED)
include_directories(${LUAJIT_INCLUDE_DIRS})

find_package(LibIntl)
if(LibIntl_FOUND)
  include_directories(${LibIntl_INCLUDE_DIR})
endif()

# Determine platform's threading library. Set CMAKE_THREAD_PREFER_PTHREAD
# explicitly to indicate a strong preference for pthread. It is an error to not
# have pthread installed even if, for example, the Win32 threading API is found.
set(CMAKE_THREAD_PREFER_PTHREAD ON)
find_package(Threads REQUIRED)
if(NOT CMAKE_USE_PTHREADS_INIT)
  message(SEND_ERROR "The pthread library must be installed on your system.")
endif(NOT CMAKE_USE_PTHREADS_INIT)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

find_program(LUA_PRG luajit)

# Need a lua interpreter for running the msgapck metadata/dispatch generator
if(NOT EXISTS ${LUA_PRG})
  find_program(LUA_PRG lua)
endif()

if(EXISTS ${LUA_PRG})
  message(STATUS "Using the lua interpreter ${LUA_PRG}")
else()
  message(FATAL_ERROR "A lua interpreter is required for building the Neovim")
endif()

execute_process(COMMAND ${LUA_PRG} -e "require('lpeg')"
                RESULT_VARIABLE LUA_LPEG_MISSING
                ERROR_QUIET)

if(${LUA_LPEG_MISSING})
  message(FATAL_ERROR
          "The 'lpeg' lua package is required for building Neovim")
endif()

execute_process(COMMAND ${LUA_PRG} -e "require('cmsgpack')"
                RESULT_VARIABLE LUA_MSGPACK_MISSING
                ERROR_QUIET)

if(${LUA_MSGPACK_MISSING})
  message(FATAL_ERROR
         "The 'cmsgpack' lua package is required for building Neovim")
endif()

add_subdirectory(config)
add_subdirectory(src)
add_subdirectory(test/includes)

find_program(BUSTED_PRG busted)

find_program(MAKE_PRG NAMES gmake make)
if(MAKE_PRG)
  execute_process(
    COMMAND "${MAKE_PRG}" --version
    OUTPUT_VARIABLE MAKE_VERSION_INFO)
  if(NOT "${OUTPUT_VARIABLE}" MATCHES ".*GNU.*")
    unset(MAKE_PRG)
  endif()
endif()
if(NOT MAKE_PRG)
  message(FATAL_ERROR "GNU Make is required to build the dependencies.")
else()
  message(STATUS "Found GNU Make at ${MAKE_PRG}")
endif()

# When using make, use the $(MAKE) variable to avoid warnings about the job
# server.
if(CMAKE_GENERATOR MATCHES "Makefiles")
  set(MAKE_PRG "$(MAKE)")
endif()

if(NOT BUSTED_OUTPUT_TYPE)
  set(BUSTED_OUTPUT_TYPE "utf_terminal")
endif()

if(BUSTED_PRG)
  get_target_property(NVIM_TEST_LIB nvim-test LOCATION)
  add_custom_target(unittest
    COMMAND ${CMAKE_COMMAND}
      -DBUSTED_PRG=${BUSTED_PRG}
      -DLUA_PRG=${LUA_PRG}
      -DWORKING_DIR=${CMAKE_CURRENT_SOURCE_DIR}
      -DNVIM_TEST_LIB=${NVIM_TEST_LIB}
      -DBUSTED_OUTPUT_TYPE=${BUSTED_OUTPUT_TYPE}
      -DTEST_DIR=${CMAKE_CURRENT_SOURCE_DIR}/test
      -DTEST_INCLUDES=${CMAKE_BINARY_DIR}/test/includes/post
      -P ${CMAKE_MODULE_PATH}/RunUnittests.cmake
    DEPENDS nvim-test unittest-headers)
endif()

# Unfortunately, the below does not work under Ninja.  Ninja doesn't use a
# pseudo-tty when launching processes, because it can put many jobs in parallel
# and eat-up all the available pseudo-ttys.  Unfortunately, that doesn't work
# well with the legacy tests.  I have a branch that converts them to run under
# CTest, but it needs a little more work.
# add_custom_target(test
#   COMMAND ${MAKE_PRG} -C ${CMAKE_CURRENT_SOURCE_DIR}/src/testdir
#     VIMPROG=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/nvim)
