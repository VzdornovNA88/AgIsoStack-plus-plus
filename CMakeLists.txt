cmake_minimum_required(VERSION 3.16)

if(ARM_NONE_EABI_PLATFORM)
set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
endif()

project(
  isobus
  VERSION 0.1
  LANGUAGES CXX
  DESCRIPTION
    "A control function focused implementation of the major ISOBUS and J1939 transport layers"
)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

if(ARM_NONE_EABI_PLATFORM)
  include(cmake/arm-none-eabi.cmake)
endif()

# Make CTest available and the BUILD_TESTING option (OFF by default)
option(BUILD_TESTING "Set to ON to enable building of tests from top level" OFF)
include(CTest)
if(BUILD_TESTING
   AND NOT MSVC
   AND NOT APPLE)
  # Set --coverage flag for gcovr (SonarCloud)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage")
endif()

if(ARM_NONE_EABI_PLATFORM)
  add_compile_options(-Wall -Wextra)
elseif(ESP_PLATFORM)
  add_compile_options(-Wall -Wextra)
elseif(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

option(DISABLE_BUSLOAD_MONITORING "Disable CAN bus load monitoring" OFF)

option(
  CAN_STACK_DISABLE_THREADS
  "Set to ON to disable multi-threading, which removes the need for std::thread, std::mutex, and related libraries."
  OFF)

if(ARM_NONE_EABI_PLATFORM AND NOT CAN_STACK_DISABLE_THREADS)
  set(CAN_STACK_DISABLE_THREADS ON CACHE BOOL "Set to ON to disable multi-threading, which removes the need for std::thread, std::mutex, and related libraries." FORCE)
endif()
if(NOT CAN_STACK_DISABLE_THREADS AND NOT ARDUINO)
  # Find packages required for Threading
  set(THREADS_PREFER_PTHREAD_FLAG ON)
  # ESP-IDF doesn't implement find_package(Threads) correctly (dec 2022)
  if(NOT ESP_PLATFORM)
    find_package(Threads REQUIRED)
  endif()
endif()

# A handy function to prepend text to all elements in a list (useful for
# subdirectories)
function(prepend var prefix)
  set(listVar "")
  foreach(arg ${ARGN})
    list(APPEND listVar "${prefix}/${arg}")
  endforeach(arg)
  set(${var}
      "${listVar}"
      PARENT_SCOPE)
endfunction(prepend)

# Add subdirectories
add_subdirectory("utility")
add_subdirectory("isobus")
add_subdirectory("hardware_integration")

option(BUILD_EXAMPLES "Set to ON to enable building of examples from top level"
       OFF)
if(BUILD_EXAMPLES)
  if(NOT ARM_NONE_EABI_PLATFORM)
    add_subdirectory("examples/transport_layer")
    add_subdirectory("examples/diagnostic_protocol")
    add_subdirectory("examples/pgn_requests")
    add_subdirectory("examples/nmea2000/fast_packet_protocol")
    add_subdirectory("examples/nmea2000/nmea2000_generator")
    add_subdirectory("examples/nmea2000/nmea2000_parser")
    add_subdirectory("examples/virtual_terminal/version3_object_pool")
    add_subdirectory("examples/virtual_terminal/aux_functions")
    add_subdirectory("examples/virtual_terminal/aux_inputs")
    add_subdirectory("examples/virtual_terminal/iop_parser_tester")
    add_subdirectory("examples/task_controller_client")
    add_subdirectory("examples/task_controller_server")
    add_subdirectory("examples/guidance")
  endif()
  add_subdirectory("examples/seeder_example")
endif()

if(BUILD_TESTING)
  add_subdirectory("test")
endif()

if(ARM_NONE_EABI_PLATFORM OR ESP_PLATFORM)
  return()
endif()

install(
  EXPORT isobusTargets
  FILE isobusTargets.cmake
  NAMESPACE isobus::
  DESTINATION lib/cmake/isobus)
configure_file(cmake/isobusConfig.cmake.in isobusConfig.cmake @ONLY)

write_basic_package_version_file(isobusConfigVersion.cmake
                                 COMPATIBILITY SameMajorVersion)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/isobusConfig.cmake"
              "${CMAKE_CURRENT_BINARY_DIR}/isobusConfigVersion.cmake"
        DESTINATION lib/cmake/isobus)

set(CPACK_PACKAGE_CONTACT "delgrossoengineering@protonmail.com")
set(CPACK_DEBIAN_FILENAME DEB-DEFAULT)
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE
    "https://github.com/Open-Agriculture/AgIsoStack-plus-plus")
set(CPACK_PACKAGE_VENDOR "Open-Agriculture")
set(CPACK_COMPONENTS_GROUPING ALL_COMPONENTS_IN_ONE)
include(CPack)
