cmake_minimum_required(VERSION 2.8.4)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules")
project(SOEM C)

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
   # Default to installing in SOEM source directory
   set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/install)
endif()

set(SOEM_INCLUDE_INSTALL_DIR include/soem)

message("CMAKE_SYSTEM_NAME is ${CMAKE_SYSTEM_NAME}")

if(WIN32)
   set(OS "win32")
   include_directories(oshw/win32/wpcap/Include)
   link_directories(${CMAKE_SOURCE_DIR}/oshw/win32/wpcap/Lib)
elseif(UNIX)
   set(OS "linux")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "rt-kernel")
   set(OS "rtk")
   message("ARCH is ${ARCH}")
   message("BSP is ${BSP}")
   include_directories(oshw/${OS}/${ARCH})
   add_definitions("-Wno-unused-but-set-variable")
   add_definitions("-Wno-unused-function")
   add_definitions("-Wno-format")
endif()

message("OS is ${OS}")

include_directories(soem)
include_directories(osal)
include_directories(osal/${OS})
include_directories(oshw/${OS})

if (MSVC)
  set(CMAKE_C_FLAGS_RELEASE "/D _CRT_SECURE_NO_WARNINGS /EHsc")
  set(CMAKE_C_FLAGS_DEBUG "/D _CRT_SECURE_NO_WARNINGS /EHsc /ZI /Od")
endif()

file(GLOB SOEM_SOURCES soem/*.c)
file(GLOB OSAL_SOURCES osal/${OS}/*.c)
file(GLOB OSHW_SOURCES oshw/${OS}/*.c)

if(WIN32)
  set(SOEM_SOURCES ${SOEM_SOURCES})
endif()

file(GLOB SOEM_HEADERS soem/*.h)
file(GLOB OSAL_HEADERS osal/osal.h osal/${OS}/*.h)
file(GLOB OSHW_HEADERS oshw/${OS}/*.h)

if(${CMAKE_SYSTEM_NAME} MATCHES "rt-kernel")
  file(GLOB OSHW_ARCHSOURCES oshw/${OS}/${ARCH}/*.c)
endif()

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
add_library(soem STATIC ${SOEM_SOURCES} ${OSAL_SOURCES} ${OSHW_SOURCES} ${OSHW_ARCHSOURCES})
if(WIN32)
  target_link_libraries(soem wpcap.lib Packet.lib Ws2_32.lib Winmm.lib ${HDF5_LIBRARIES})
elseif(UNIX)
  target_link_libraries(soem pthread rt)
endif()

install(TARGETS soem DESTINATION lib)
install(FILES ${SOEM_HEADERS} ${OSAL_HEADERS} ${OSHW_HEADERS} DESTINATION ${SOEM_INCLUDE_INSTALL_DIR})

add_subdirectory(test/linux/slaveinfo)
add_subdirectory(test/linux/eepromtool)
add_subdirectory(test/linux/simple_test)



find_package(Git QUIET)
if(GIT_FOUND)
    # output format <tag>-<commit-id>-dirty or <tag>-<commit-id> or <commit-id>
    execute_process(COMMAND ${GIT_EXECUTABLE} describe --long --tags --dirty --always
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        RESULT_VARIABLE  GIT_EXIT_STATUS
        OUTPUT_VARIABLE SOEM_PACKAGE_GIT_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE)
    if(NOT GIT_EXIT_STATUS EQUAL 0)
        message(WARNING "no git repository or unable to run git describe")
        set(SIMOX_PACKAGE_GIT_VERSION "Unknown")
    else()
        message(STATUS "SOEM git version: ${SOEM_PACKAGE_GIT_VERSION}")
    endif()
endif()

string(REGEX REPLACE "^v" "" SOEM_PACKAGE_GIT_VERSION ${SOEM_PACKAGE_GIT_VERSION})
string(REGEX REPLACE "-.*$" "" SOEM_PACKAGE_GIT_VERSION ${SOEM_PACKAGE_GIT_VERSION})
string(REPLACE "." ";" SOEM_PACKAGE_GIT_VERSION_LIST ${SOEM_PACKAGE_GIT_VERSION})
list(GET SOEM_PACKAGE_GIT_VERSION_LIST 0 CPACK_PACKAGE_VERSION_MAJOR)
list(GET SOEM_PACKAGE_GIT_VERSION_LIST 1 CPACK_PACKAGE_VERSION_MINOR)
list(GET SOEM_PACKAGE_GIT_VERSION_LIST 2 CPACK_PACKAGE_VERSION_PATCH)

message(STATUS "package version is ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")

set(CPACK_GENERATOR "DEB")
set(CPACK_PACKAGE_CONTACT "Open EtherCAT Society")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
include(CPack)
