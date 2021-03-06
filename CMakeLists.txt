CMAKE_MINIMUM_REQUIRED(VERSION 3.5 FATAL_ERROR)

INCLUDE(GNUInstallDirs)

# ---[ Project
PROJECT(pthreadpool C CXX)

# ---[ Options.
SET(PTHREADPOOL_LIBRARY_TYPE "default" CACHE STRING "Type of library (shared, static, or default) to build")
SET_PROPERTY(CACHE PTHREADPOOL_LIBRARY_TYPE PROPERTY STRINGS default static shared)
OPTION(PTHREADPOOL_ALLOW_DEPRECATED_API "Enable deprecated API functions" ON)
IF("${CMAKE_SOURCE_DIR}" STREQUAL "${PROJECT_SOURCE_DIR}")
  OPTION(PTHREADPOOL_BUILD_TESTS "Build pthreadpool unit tests" ON)
  OPTION(PTHREADPOOL_BUILD_BENCHMARKS "Build pthreadpool micro-benchmarks" ON)
ELSE()
  SET(PTHREADPOOL_BUILD_TESTS OFF CACHE BOOL "Build pthreadpool unit tests")
  SET(PTHREADPOOL_BUILD_BENCHMARKS OFF CACHE BOOL "Build pthreadpool micro-benchmarks")
ENDIF()

# ---[ CMake options
IF(PTHREADPOOL_BUILD_TESTS)
  ENABLE_TESTING()
ENDIF()

MACRO(PTHREADPOOL_TARGET_ENABLE_CXX11 target)
  SET_TARGET_PROPERTIES(${target} PROPERTIES
    CXX_STANDARD 11
    CXX_EXTENSIONS NO)
ENDMACRO()

# ---[ Download deps
IF(NOT DEFINED FXDIV_SOURCE_DIR)
  MESSAGE(STATUS "Downloading FXdiv to ${CMAKE_BINARY_DIR}/FXdiv-source (define FXDIV_SOURCE_DIR to avoid it)")
  CONFIGURE_FILE(cmake/DownloadFXdiv.cmake "${CMAKE_BINARY_DIR}/FXdiv-download/CMakeLists.txt")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/FXdiv-download")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" --build .
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/FXdiv-download")
  SET(FXDIV_SOURCE_DIR "${CMAKE_BINARY_DIR}/FXdiv-source" CACHE STRING "FXdiv source directory")
ENDIF()

IF(PTHREADPOOL_BUILD_TESTS AND NOT DEFINED GOOGLETEST_SOURCE_DIR)
  MESSAGE(STATUS "Downloading Google Test to ${CMAKE_BINARY_DIR}/googletest-source (define GOOGLETEST_SOURCE_DIR to avoid it)")
  CONFIGURE_FILE(cmake/DownloadGoogleTest.cmake "${CMAKE_BINARY_DIR}/googletest-download/CMakeLists.txt")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/googletest-download")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" --build .
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/googletest-download")
  SET(GOOGLETEST_SOURCE_DIR "${CMAKE_BINARY_DIR}/googletest-source" CACHE STRING "Google Test source directory")
ENDIF()

IF(PTHREADPOOL_BUILD_BENCHMARKS AND NOT DEFINED GOOGLEBENCHMARK_SOURCE_DIR)
  MESSAGE(STATUS "Downloading Google Benchmark to ${CMAKE_BINARY_DIR}/googlebenchmark-source (define GOOGLEBENCHMARK_SOURCE_DIR to avoid it)")
  CONFIGURE_FILE(cmake/DownloadGoogleBenchmark.cmake "${CMAKE_BINARY_DIR}/googlebenchmark-download/CMakeLists.txt")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/googlebenchmark-download")
  EXECUTE_PROCESS(COMMAND "${CMAKE_COMMAND}" --build .
    WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/googlebenchmark-download")
  SET(GOOGLEBENCHMARK_SOURCE_DIR "${CMAKE_BINARY_DIR}/googlebenchmark-source" CACHE STRING "Google Benchmark source directory")
ENDIF()

# ---[ pthreadpool library
IF(PTHREADPOOL_ALLOW_DEPRECATED_API)
  SET(PTHREADPOOL_SRCS src/threadpool-legacy.c)
ENDIF()
IF(CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
  LIST(APPEND PTHREADPOOL_SRCS src/threadpool-shim.c)
ELSE()
  LIST(APPEND PTHREADPOOL_SRCS src/threadpool-pthreads.c)
ENDIF()

IF(${CMAKE_VERSION} VERSION_LESS "3.0")
  ADD_LIBRARY(pthreadpool_interface STATIC include/pthreadpool.h)
  SET_TARGET_PROPERTIES(pthreadpool_interface PROPERTIES LINKER_LANGUAGE C)
ELSE()
  ADD_LIBRARY(pthreadpool_interface INTERFACE)
ENDIF()
TARGET_INCLUDE_DIRECTORIES(pthreadpool_interface INTERFACE include)
IF(NOT PTHREADPOOL_ALLOW_DEPRECATED_API)
  TARGET_COMPILE_DEFINITIONS(pthreadpool_interface INTERFACE PTHREADPOOL_NO_DEPRECATED_API=1)
ENDIF()
INSTALL(FILES include/pthreadpool.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

IF(PTHREADPOOL_LIBRARY_TYPE STREQUAL "default")
  ADD_LIBRARY(pthreadpool ${PTHREADPOOL_SRCS})
ELSEIF(PTHREADPOOL_LIBRARY_TYPE STREQUAL "shared")
  ADD_LIBRARY(pthreadpool SHARED ${PTHREADPOOL_SRCS})
ELSEIF(PTHREADPOOL_LIBRARY_TYPE STREQUAL "static")
  ADD_LIBRARY(pthreadpool STATIC ${PTHREADPOOL_SRCS})
ELSE()
  MESSAGE(FATAL_ERROR "Unsupported library type ${PTHREADPOOL_LIBRARY_TYPE}")
ENDIF()

SET_TARGET_PROPERTIES(pthreadpool PROPERTIES
  C_STANDARD 11
  C_EXTENSIONS NO)
TARGET_LINK_LIBRARIES(pthreadpool PUBLIC pthreadpool_interface)
TARGET_INCLUDE_DIRECTORIES(pthreadpool PRIVATE src)
IF(NOT CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
  SET(CMAKE_THREAD_PREFER_PTHREAD TRUE)
  IF(NOT CMAKE_GENERATOR STREQUAL "Xcode")
    FIND_PACKAGE(Threads REQUIRED)
  ENDIF()
  IF(CMAKE_USE_PTHREADS_INIT)
    IF(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME STREQUAL "Android")
      TARGET_COMPILE_OPTIONS(pthreadpool PUBLIC -pthread)
    ENDIF()
  ENDIF()
  TARGET_LINK_LIBRARIES(pthreadpool PUBLIC ${CMAKE_THREAD_LIBS_INIT})
ENDIF()
IF(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  TARGET_COMPILE_DEFINITIONS(pthreadpool PRIVATE _GNU_SOURCE=1)
ENDIF()

# ---[ Configure FXdiv
IF(NOT TARGET fxdiv)
  SET(FXDIV_BUILD_TESTS OFF CACHE BOOL "")
  SET(FXDIV_BUILD_BENCHMARKS OFF CACHE BOOL "")
  ADD_SUBDIRECTORY(
    "${FXDIV_SOURCE_DIR}"
    "${CMAKE_BINARY_DIR}/FXdiv")
ENDIF()
TARGET_LINK_LIBRARIES(pthreadpool PRIVATE fxdiv)

INSTALL(TARGETS pthreadpool
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

IF(PTHREADPOOL_BUILD_TESTS)
  # ---[ Build google test
  IF(NOT TARGET gtest)
    SET(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    ADD_SUBDIRECTORY(
      "${GOOGLETEST_SOURCE_DIR}"
      "${CMAKE_BINARY_DIR}/googletest")
  ENDIF()

  ADD_EXECUTABLE(pthreadpool-test test/pthreadpool.cc)
  SET_TARGET_PROPERTIES(pthreadpool-test PROPERTIES
    CXX_STANDARD 11
    CXX_EXTENSIONS NO)
  TARGET_LINK_LIBRARIES(pthreadpool-test pthreadpool gtest gtest_main)
  ADD_TEST(pthreadpool pthreadpool-test)
ENDIF()

IF(PTHREADPOOL_BUILD_BENCHMARKS)
  # ---[ Build google benchmark
  IF(NOT TARGET benchmark)
    SET(BENCHMARK_ENABLE_TESTING OFF CACHE BOOL "")
    ADD_SUBDIRECTORY(
      "${GOOGLEBENCHMARK_SOURCE_DIR}"
      "${CMAKE_BINARY_DIR}/googlebenchmark")
  ENDIF()

  ADD_EXECUTABLE(latency-bench bench/latency.cc)
  SET_TARGET_PROPERTIES(latency-bench PROPERTIES
    CXX_STANDARD 11
    CXX_EXTENSIONS NO)
  TARGET_LINK_LIBRARIES(latency-bench pthreadpool benchmark)

  ADD_EXECUTABLE(throughput-bench bench/throughput.cc)
  SET_TARGET_PROPERTIES(throughput-bench PROPERTIES
    CXX_STANDARD 11
    CXX_EXTENSIONS NO)
  TARGET_LINK_LIBRARIES(throughput-bench pthreadpool benchmark)
ENDIF()
