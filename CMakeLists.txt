cmake_minimum_required(VERSION 3.13.0)
project(REINDEER VERSION 1.0.2 LANGUAGES CXX C)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

option(NATIVE "Build with -march=native" ON)

if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)

find_package(ZLIB REQUIRED)
find_package(OpenMP REQUIRED)

add_library(external INTERFACE)
target_link_libraries(external INTERFACE ZLIB::ZLIB OpenMP::OpenMP_CXX)

add_library(build_type_flags INTERFACE)
target_compile_options(build_type_flags INTERFACE
  $<$<CONFIG:Debug>:-O0 -g -fstrict-aliasing>
  $<$<CONFIG:Release>:-O3 -DNDEBUG -fstrict-aliasing>
  $<$<CONFIG:Profile>:-O3 -ggdb3 -DNDEBUG -fno-inline>
)

include(CheckIPOSupported)
add_library(lto_opti INTERFACE)
check_ipo_supported(RESULT res_ipo OUTPUT out_ipo)
if (res_ipo)
  set_property(TARGET lto_opti PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
  target_link_libraries(build_type_flags INTERFACE lto_opti)
endif()

if (NATIVE)
  target_compile_options(build_type_flags INTERFACE "-march=native")
endif()

add_library(headers INTERFACE)
target_include_directories(headers INTERFACE
  ${CMAKE_SOURCE_DIR}/src
  ${CMAKE_SOURCE_DIR}/blight
  ${CMAKE_SOURCE_DIR}/blight/lz4
)

set(BDIR ${CMAKE_SOURCE_DIR}/blight)
file(GLOB_RECURSE SOURCES "src/*.cpp")

add_executable(${PROJECT_NAME}
  ${SOURCES}
  ${CMAKE_SOURCE_DIR}/main.cpp
  ${BDIR}/trlec.c
  ${BDIR}/trled.c
  ${BDIR}/blight.cpp
  ${BDIR}/utils.cpp
  ${BDIR}/lz4/xxhash.c)

target_link_libraries(${PROJECT_NAME} build_type_flags external headers lz4)

include(ExternalProject)
ExternalProject_Add(
  BCALM2
  PREFIX BCALM2
  SOURCE_DIR ${CMAKE_SOURCE_DIR}/bcalm2
  EXCLUDE_FROM_ALL
  LOG_CONFIGURE ON
  LOG_BUILD ON
  LOG_INSTALL ON
  BUILD_COMMAND make
  INSTALL_COMMAND ""
)

add_custom_command(
  TARGET BCALM2 POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/BCALM2/src/BCALM2-build/bcalm ${CMAKE_SOURCE_DIR}/bin/bcalm)



