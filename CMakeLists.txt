# cmake_minimum_required(VERSION <specify CMake version here>)
project(Temp C)

set(CMAKE_C_STANDARD 11)

include_directories(include)

add_executable(Temp
        include/util.h
        main.c)
