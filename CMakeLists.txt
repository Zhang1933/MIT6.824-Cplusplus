cmake_minimum_required (VERSION 3.16)
project (mit6824 VERSION 1.0)

find_package (glog 0.8.0 REQUIRED)
find_package(fmt REQUIRED)

set(CMAKE_BUILD_TYPE Debug)

# add_compile_options(-fsanitize=address)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")


set(CMAKE_VERBOSE_MAKEFILE ON)
set(TARGETSRC ${CMAKE_SOURCE_DIR}/src)
# Thrift
find_program(THRIFT_COMPILER thrift)
if(NOT THRIFT_COMPILER)
    message(FATAL_ERROR "Thrift compiler not found. Please install Thrift or add it to your PATH.")
endif()
SET(ThriftOutputDir ${CMAKE_CURRENT_BINARY_DIR}/thrift/gen/)

# KVRaft thrift
SET(ThriftOutputDirKVRaft ${ThriftOutputDir}/KVRaft/)
FILE(MAKE_DIRECTORY ${ThriftOutputDir}/KVRaft)
set(KVRaft "${TARGETSRC}/rpc/KVRaft.thrift")
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${KVRaft}")
execute_process(
    COMMAND ${THRIFT_COMPILER} -r --gen cpp -out ${ThriftOutputDirKVRaft} ${TARGETSRC}/rpc/KVRaft.thrift
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    INPUT_FILE "${KVRaft}"
)
file(GLOB generated_KVRaft "${ThriftOutputDirKVRaft}/*")
list(FILTER generated_KVRaft EXCLUDE REGEX ".*skeleton\\.cpp$")
ADD_LIBRARY(kvraft-rpc SHARED  ${generated_KVRaft})

# MapReduce thrift
SET(ThriftOutputDirMapReduce ${CMAKE_CURRENT_BINARY_DIR}/thrift/gen/MapReduce/)
FILE(MAKE_DIRECTORY ${ThriftOutputDirMapReduce})
set(MapReduce "${TARGETSRC}/rpc/MapReduce.thrift")
set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS "${MapReduce}")
execute_process(
    COMMAND ${THRIFT_COMPILER} -r --gen cpp -out ${ThriftOutputDirMapReduce} ${TARGETSRC}/rpc/MapReduce.thrift
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    INPUT_FILE "${MapReduce}"
)
file(GLOB generated_MapReduce "${ThriftOutputDirMapReduce}/*")
list(FILTER generated_MapReduce EXCLUDE REGEX ".*skeleton\\.cpp$")
ADD_LIBRARY(mr-rpc SHARED  ${generated_MapReduce})

include_directories(${ThriftOutputDirMapReduce}
                    ${ThriftOutputDirKVRaft}
                    ${CMAKE_SOURCE_DIR}/include)
link_libraries(-lthrift -lgflags glog::glog fmt::fmt kvraft-rpc mr-rpc)

macro(buildelf)
  set(singleValue NAME)
  set(multipleValues SOURCES)
  cmake_parse_arguments(local "" "${singleValue}" "${multipleValues}" ${ARGN})
  add_executable(${local_NAME} ${local_SOURCES})
endmacro()
macro(buillibso)
  set(singleValue NAME)
  set(multipleValues SOURCES)
  cmake_parse_arguments(local "" "${singleValue}" "${multipleValues}" ${ARGN})
  ADD_LIBRARY(${local_NAME} SHARED ${local_SOURCES})
endmacro()

# master
buildelf(
    NAME master
    SOURCES ${TARGETSRC}/MapReduce/master.cpp
)
# worker
buildelf(
    NAME worker
    SOURCES ${TARGETSRC}/MapReduce/worker.cpp
)
# wordcount
buillibso(
    NAME WordCount
    SOURCES "${TARGETSRC}/MapReduce/user-program/WordCount.cpp"
)

