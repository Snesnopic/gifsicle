cmake_minimum_required(VERSION 3.15)
project(gifsicle VERSION 1.96 LANGUAGES C)

# build options
option(BUILD_GIFVIEW "build gifview x11 viewer" ON)
option(BUILD_GIFDIFF "build gifdiff utility" ON)
option(ENABLE_UNGIF "build without lzw compression" OFF)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

include(CheckIncludeFiles)
# check headers dynamically
check_include_files(pthread.h HAVE_PTHREAD_H)
check_include_files(unistd.h HAVE_UNISTD_H)
check_include_files(sys/types.h HAVE_SYS_TYPES_H)

# global macros (replaces config.h.in)
add_compile_definitions(
        VERSION="${PROJECT_VERSION}"
        PACKAGE="${PROJECT_NAME}"
        GIF_ALLOCATOR_DEFINED=1
        PATHNAME_SEPARATOR='/'
        RANDOM=random
        HAVE_INTTYPES_H=1
        HAVE_SNPRINTF=1
        HAVE_STRERROR=1
        HAVE_POW=1
        HAVE_UINTPTR_T=1
)

# add definitions only if headers are found
if(HAVE_UNISTD_H)
    add_compile_definitions(HAVE_UNISTD_H=1)
endif()

if(HAVE_SYS_TYPES_H)
    add_compile_definitions(HAVE_SYS_TYPES_H=1)
endif()

# thread support: enable only if posix threads are present
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads)
if(Threads_FOUND AND HAVE_PTHREAD_H)
    add_compile_definitions(
            ENABLE_THREADS=1
            HAVE___SYNC_ADD_AND_FETCH=1
    )
endif()

# generate config.h for legacy includes
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/config.h" "/* auto-generated config */
#ifndef CONFIG_H
#define CONFIG_H
#if defined(_WIN32) || defined(_WIN64)
# include <fcntl.h>
# include <io.h>
# define popen _popen
# define pclose _pclose
#endif
#endif
")
include_directories("${CMAKE_CURRENT_BINARY_DIR}" "include")

# libgifsicle (core library)
set(LIBGIFSICLE_SOURCES
        src/clp.c
        src/fmalloc.c
        src/giffunc.c
        src/gifread.c
        src/gifunopt.c
        src/kcolor.c
        src/merge.c
        src/optimize.c
        src/quantize.c
        src/support.c
        src/xform.c
)

if(ENABLE_UNGIF)
    list(APPEND LIBGIFSICLE_SOURCES src/ungifwrt.c)
    add_compile_definitions(GIF_UNGIF=1)
else()
    list(APPEND LIBGIFSICLE_SOURCES src/gifwrite.c)
endif()

# msvc already includes math functions in the standard c runtime
if(MSVC)
    set(MATH_LIB "")
else()
    set(MATH_LIB "m")
endif()

add_library(libgifsicle STATIC ${LIBGIFSICLE_SOURCES})
target_link_libraries(libgifsicle PUBLIC ${MATH_LIB})

if(Threads_FOUND AND HAVE_PTHREAD_H)
    target_link_libraries(libgifsicle PUBLIC Threads::Threads)
endif()

# main cli target
add_executable(gifsicle src/gifsicle.c)
target_link_libraries(gifsicle PRIVATE libgifsicle)

# gifdiff target
if(BUILD_GIFDIFF)
    add_executable(gifdiff src/gifdiff.c)
    target_link_libraries(gifdiff PRIVATE libgifsicle)
endif()

# gifview target
if(BUILD_GIFVIEW)
    find_package(X11)
    if(X11_FOUND)
        add_executable(gifview src/gifview.c src/gifx.c)
        target_link_libraries(gifview PRIVATE libgifsicle X11::X11)
    else()
        message(WARNING "x11 not found, skipping gifview target")
    endif()
endif()

# enable ctest framework
enable_testing()

add_custom_target(check
        COMMAND ${CMAKE_CTEST_COMMAND} --force-new-ctest-process --output-on-failure
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

file(GLOB TEST_FILES "test/*.testie")
foreach(TEST_FILE ${TEST_FILES})
    get_filename_component(TEST_NAME ${TEST_FILE} NAME_WE)
    add_test(NAME ${TEST_NAME} COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/test/testie ${TEST_FILE})
    set_tests_properties(${TEST_NAME} PROPERTIES
            ENVIRONMENT "PATH=${CMAKE_CURRENT_BINARY_DIR}:$ENV{PATH}"
    )
endforeach()