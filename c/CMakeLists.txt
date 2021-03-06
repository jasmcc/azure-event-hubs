#Copyright (c) Microsoft. All rights reserved.
#Licensed under the MIT license. See LICENSE file in the project root for full license information.

cmake_minimum_required(VERSION 2.8.11)
project(eventhub_client)

#the following variables are project-wide and can be used with cmake-gui
option(run_e2e_tests "set run_e2e_tests to ON to run e2e tests (default is OFF) [if possible, they are always build]" OFF)
option(skip_unittests "set skip_unittests to ON to skip unittests (default is OFF)[if possible, they are always build]" OFF)
option(compileOption_C "passes a string to the command line of the C compiler" OFF)
option(compileOption_CXX "passes a string to the command line of the C++ compiler" OFF)

#Use solution folders. 
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

add_definitions(-DREFCOUNT_ATOMIC_DONTCARE)
add_definitions(-D__STDC_NO_ATOMICS__=1)

if(IN_OPENWRT)
    ADD_DEFINITIONS("$ENV{TARGET_LDFLAGS}" "$ENV{TARGET_CPPFLAGS}" "$ENV{TARGET_CFLAGS}")
    INCLUDE_DIRECTORIES("$ENV{TOOLCHAIN_DIR}/usr/include" "$ENV{TARGET_LDFLAGS}" "$ENV{TARGET_CPPFLAGS}" "$ENV{TARGET_CFLAGS}")
endif()

add_subdirectory(azure-uamqp-c)

include_directories(${UAMQP_INC_FOLDER})

if(NOT TARGET aziotsharedutil)
    add_subdirectory(azure-c-shared-utility/c)
endif()

enable_testing()

#if any compiler has a command line switch called "OFF" then it will need special care
if(NOT "${compileOption_C}" STREQUAL "OFF")
    set(CMAKE_C_FLAGS "${compileOption_C} ${CMAKE_C_FLAGS}")
endif()

if(NOT "${compileOption_CXX}" STREQUAL "OFF")
    set(CMAKE_CXX_FLAGS "${compileOption_CXX} ${CMAKE_CXX_FLAGS}")
endif()


#this project uses several other projects that are build not by these CMakeFiles
#this project also targets several OSes

function(linkSharedUtil whatIsBuilding)
    target_link_libraries(${whatIsBuilding} aziotsharedutil)
endfunction(linkSharedUtil)

macro(compileAsC99)
  if (CMAKE_VERSION VERSION_LESS "3.1")
    if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
      set (CMAKE_C_FLAGS "--std=c99 ${CMAKE_C_FLAGS}")
    if (NOT IN_OPENWRT)
      set (CMAKE_CXX_FLAGS "--std=c++11 ${CMAKE_CXX_FLAGS}")
    endif()
    endif()
  else()
    set (CMAKE_C_STANDARD 99)
    set (CMAKE_CXX_STANDARD 11)
  endif()
endmacro(compileAsC99)

macro(compileAsC11)
  if (CMAKE_VERSION VERSION_LESS "3.1")
    if (CMAKE_C_COMPILER_ID STREQUAL "GNU")
      set (CMAKE_C_FLAGS "--std=c11 ${CMAKE_C_FLAGS}")
      set (CMAKE_C_FLAGS "-D_POSIX_C_SOURCE=200112L ${CMAKE_C_FLAGS}")
      set (CMAKE_CXX_FLAGS "--std=c++11 ${CMAKE_CXX_FLAGS}")
    endif()
  else()
    set (CMAKE_C_STANDARD 11)
    set (CMAKE_CXX_STANDARD 11)
  endif()
endmacro(compileAsC11)

function(windows_unittests_add_dll whatIsBuilding)	
    link_directories(${whatIsBuilding}_dll ${MICROMOCK_UNITTEST_LIB_DIR} ${CTEST_LIB_DIR} ${SHARED_UTIL_LIB_DIR} $ENV{VCInstallDir}UnitTest/lib)
    
    add_library(${whatIsBuilding}_dll SHARED 
        ${${whatIsBuilding}_cpp_files} 
        ${${whatIsBuilding}_h_files} 
        ${${whatIsBuilding}_c_files}
    )
    
    #Add UnitTests to their own folder
    set_target_properties(${whatIsBuilding}_dll
               PROPERTIES
               FOLDER "UnitTests") 
               
    target_include_directories(${whatIsBuilding}_dll PUBLIC ${sharedutil_include_directories} $ENV{VCInstallDir}UnitTest/include)
    target_compile_definitions(${whatIsBuilding}_dll PUBLIC -DCPP_UNITTEST)
    target_link_libraries(${whatIsBuilding}_dll micromock_cpp_unittest ctest ${ARGN})
endfunction()

function(windows_unittests_add_exe whatIsBuilding)
    link_directories(${whatIsBuilding}_exe ${MICROMOCK_UNITTEST_LIB_DIR} ${CTEST_LIB_DIR} ${SHARED_UTIL_LIB_DIR} $ENV{VCInstallDir}UnitTest/lib)
    
    add_executable(${whatIsBuilding}_exe
        ${${whatIsBuilding}_cpp_files} 
        ${${whatIsBuilding}_h_files} 
        ${${whatIsBuilding}_c_files}
        ${CMAKE_CURRENT_LIST_DIR}/main.c
    )
    
    #Add UnitTests to their own folder
    set_target_properties(${whatIsBuilding}_exe
               PROPERTIES
               FOLDER "UnitTests")
    
    target_compile_definitions(${whatIsBuilding}_exe PUBLIC -DUSE_CTEST)
    target_include_directories(${whatIsBuilding}_exe PUBLIC ${sharedutil_include_directories})
    target_link_libraries(${whatIsBuilding}_exe micromock_ctest ctest ${ARGN})
    add_test(NAME ${whatIsBuilding} COMMAND ${whatIsBuilding}_exe)
endfunction()

function(windows_unittests_add_lib whatIsBuilding)	
    link_directories(${whatIsBuilding}_lib ${MICROMOCK_UNITTEST_LIB_DIR} ${CTEST_LIB_DIR} ${SHARED_UTIL_LIB_DIR} $ENV{VCInstallDir}UnitTest/lib)
    
    add_library(${whatIsBuilding}_lib STATIC 
        ${${whatIsBuilding}_cpp_files} 
        ${${whatIsBuilding}_h_files} 
        ${${whatIsBuilding}_c_files}
    )
    
        #Add UnitTests to their own folder
    set_target_properties(${whatIsBuilding}_lib
               PROPERTIES
               FOLDER "UnitTests")
               
    target_include_directories(${whatIsBuilding}_lib PUBLIC ${sharedutil_include_directories})
    target_compile_definitions(${whatIsBuilding}_lib PUBLIC -DUSE_CTEST)
    target_link_libraries(${whatIsBuilding}_lib micromock_ctest ctest ${ARGN})
endfunction()

function(build_test_artifacts whatIsBuilding use_gballoc)
    
    #the first argument is what is building
    #the second argument is whether the tests should be build with gballoc #defines or not
    #the following arguments are a list of libraries to link with
    
    if(${use_gballoc})
        add_definitions(-DGB_MEASURE_MEMORY_FOR_THIS -DGB_DEBUG_ALLOC)
    else()	
    endif()
    
    #setting #defines
    if(WIN32)
        add_definitions(-D_CRT_SECURE_NO_WARNINGS)		
    else()
    endif()
    
    #setting includes
    set(sharedutil_include_directories ${MICROMOCK_INC_FOLDER} ${TESTRUNNERSWITCHER_INC_FOLDER} ${CTEST_INC_FOLDER} ${SAL_INC_FOLDER} ${SHARED_UTIL_INC_FOLDER} ${SHARED_UTIL_SRC_FOLDER})
    if(WIN32)				
    else()
        include_directories(${sharedutil_include_directories})
    endif()

    #setting output type
    if(WIN32)
        #to disable running e2e or longhaul or unittests tests for windows, we build the the same thing as "static library" so it is not picked up by visual studio
        if(
            (("${whatIsBuilding}" MATCHES ".*e2e.*") AND NOT ${run_e2e_tests}) OR
            (("${whatIsBuilding}" MATCHES ".*longhaul.*") AND NOT ${run_longhaul_tests}) OR
            (("${whatIsBuilding}" MATCHES ".*unittests.*") AND ${skip_unittests})
        )
            windows_unittests_add_lib(${whatIsBuilding} ${ARGN})
        else()
            windows_unittests_add_exe(${whatIsBuilding} ${ARGN})
            windows_unittests_add_dll(${whatIsBuilding} ${ARGN})
        endif()
    else()
        if(
            (("${whatIsBuilding}" MATCHES ".*e2e.*") AND NOT ${run_e2e_tests}) OR
            (("${whatIsBuilding}" MATCHES ".*longhaul.*") AND NOT ${run_longhaul_tests}) OR
            (("${whatIsBuilding}" MATCHES ".*unittests.*") AND ${skip_unittests})
        )
            windows_unittests_add_lib(${whatIsBuilding} ${ARGN})
        else()
            windows_unittests_add_exe(${whatIsBuilding} ${ARGN})
        endif()
    endif()
    
endfunction(build_test_artifacts)

add_subdirectory(testtools)

set(SHARED_UTIL_ADAPTER_FOLDER "${CMAKE_CURRENT_LIST_DIR}/azure-c-shared-utility/c/adapters")
if(WIN32)
    set(LOCK_C_FILE ${SHARED_UTIL_ADAPTER_FOLDER}/lock_c11.c)
    set(THREAD_C_FILE ${SHARED_UTIL_ADAPTER_FOLDER}/threadapi_c11.c)
else()
    set(LOCK_C_FILE ${SHARED_UTIL_ADAPTER_FOLDER}/lock_pthreads.c)
    set(THREAD_C_FILE ${SHARED_UTIL_ADAPTER_FOLDER}/threadapi_pthreads.c)
endif()

add_subdirectory(eventhub_client)
