# Author: enzo418
# Description: Simple rewrite of the Makefile present here. Exports the
#              uSockets without SSL library as "usocketslib".
# ----
cmake_minimum_required(VERSION 3.12)

project(uSockets VERSION 1.0.0)

file(GLOB SRCS 
        ${CMAKE_CURRENT_LIST_DIR}/src/*.c 
        ${CMAKE_CURRENT_LIST_DIR}/src/eventing/*.c 
        ${CMAKE_CURRENT_LIST_DIR}/src/crypto/*.c
        ${CMAKE_CURRENT_LIST_DIR}/src/crypto/*.cpp
)

include_directories(${CMAKE_CURRENT_LIST_DIR}/src)
include_directories(${CMAKE_CURRENT_LIST_DIR}/uSockets/src)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3")

# By default we build the uSockets.a static library
add_library(usocketslib ${SRCS})

target_link_libraries(usocketslib "-lssl" "-lcrypto" "-lstdc++")

# LIBUS_NO_SSL
target_compile_definitions(usocketslib PUBLIC LIBUS_USE_OPENSSL)

set (CMAKE_CXX_STANDARD 17)

# clangd
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)