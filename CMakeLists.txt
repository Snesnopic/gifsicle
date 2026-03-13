cmake_minimum_required(VERSION 3.15)
project(gifsicle VERSION 1.96 LANGUAGES C)

# options
option(BUILD_GIFVIEW "build gifview x11 viewer" ON)
option(BUILD_GIFDIFF "build gifdiff utility" ON)
option(ENABLE_UNGIF "build without lzw compression" OFF)

# define macros globally instead of using config.h.in
add_compile_definitions(
        VERSION="${PROJECT_VERSION}"
        PACKAGE="${PROJECT_NAME}"
        GIF_ALLOCATOR_DEFINED=1
        PATHNAME_SEPARATOR='/'
        RANDOM=random
        HAVE_INTTYPES_H=1
        HAVE_UNISTD_H=1
        HAVE_SYS_TYPES_H=1
        HAVE_SNPRINTF=1
        HAVE_STRERROR=1
        HAVE_POW=1
)

add_compile_options(-W -Wall)

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/config.h" "/* auto-generated empty config */\n")
include_directories("${CMAKE_CURRENT_BINARY_DIR}" "include")

# libgifsicle
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

add_library(libgifsicle STATIC ${LIBGIFSICLE_SOURCES})
target_link_libraries(libgifsicle PUBLIC m)

# thread support
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads)
if(Threads_FOUND)
    target_link_libraries(libgifsicle PUBLIC Threads::Threads)
    target_compile_definitions(libgifsicle PUBLIC
            ENABLE_THREADS=1
            HAVE___SYNC_ADD_AND_FETCH=1
    )
endif()

# gifsicle executable
add_executable(gifsicle src/gifsicle.c)
target_link_libraries(gifsicle PRIVATE libgifsicle)

# gifdiff executable
if(BUILD_GIFDIFF)
    add_executable(gifdiff src/gifdiff.c)
    target_link_libraries(gifdiff PRIVATE libgifsicle)
endif()

# gifview executable
if(BUILD_GIFVIEW)
    find_package(X11)
    if(X11_FOUND)
        add_executable(gifview src/gifview.c src/gifx.c)
        target_link_libraries(gifview PRIVATE libgifsicle X11::X11)
    else()
        message(WARNING "x11 not found, skipping gifview target")
    endif()
endif()