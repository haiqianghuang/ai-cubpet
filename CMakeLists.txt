cmake_minimum_required(VERSION 3.16)
project(ai_cubpet LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

get_filename_component(REPO_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../../.." ABSOLUTE)
set(AI_CUBPET_PREFIX_HINTS
    ${CMAKE_INSTALL_PREFIX}
    ${CMAKE_PREFIX_PATH}
)
if(DEFINED ENV{PREFIX} AND NOT "$ENV{PREFIX}" STREQUAL "")
    list(APPEND AI_CUBPET_PREFIX_HINTS "$ENV{PREFIX}")
endif()
list(APPEND AI_CUBPET_PREFIX_HINTS "${REPO_ROOT}/output/staging")

find_package(Components QUIET)
find_package(Threads REQUIRED)
find_package(PkgConfig QUIET)
if(PkgConfig_FOUND)
    pkg_check_modules(LIBGPIOD QUIET libgpiod)
endif()
if(NOT LIBGPIOD_FOUND)
    find_library(LIBGPIOD_LIBRARY NAMES gpiod)
    if(NOT LIBGPIOD_LIBRARY)
        message(FATAL_ERROR "Required dependency not found: libgpiod")
    endif()
endif()

function(ai_cubpet_resolve_component out_var name)
    if(TARGET components::${name})
        set(${out_var} components::${name} PARENT_SCOPE)
        return()
    endif()

    find_library(AI_CUBPET_${name}_LIB
        NAMES ${name}
        HINTS ${AI_CUBPET_PREFIX_HINTS}
        PATH_SUFFIXES lib
        NO_DEFAULT_PATH
    )
    if(NOT AI_CUBPET_${name}_LIB)
        message(FATAL_ERROR "Required component library not found: ${name}")
    endif()

    get_filename_component(_lib_dir "${AI_CUBPET_${name}_LIB}" DIRECTORY)
    get_filename_component(_prefix "${_lib_dir}/.." ABSOLUTE)

    add_library(ai_cubpet_${name} UNKNOWN IMPORTED)
    set_target_properties(ai_cubpet_${name} PROPERTIES
        IMPORTED_LOCATION "${AI_CUBPET_${name}_LIB}"
        INTERFACE_INCLUDE_DIRECTORIES "${_prefix}/include"
    )
    set(${out_var} ai_cubpet_${name} PARENT_SCOPE)
endfunction()

ai_cubpet_resolve_component(MISC_IO_TARGET misc_io)
ai_cubpet_resolve_component(PM_TARGET pm)
ai_cubpet_resolve_component(WIFI_TARGET wifi)
ai_cubpet_resolve_component(NFC_TARGET nfc)
ai_cubpet_resolve_component(MOTOR_TARGET motor)
ai_cubpet_resolve_component(LIGHT_SENSOR_TARGET light_sensor)

find_library(STK_ALPS_LIB
    NAMES stk_alps_lib
    HINTS ${AI_CUBPET_PREFIX_HINTS}
    PATH_SUFFIXES lib
    NO_DEFAULT_PATH
)
if(NOT STK_ALPS_LIB)
    message(FATAL_ERROR "Required W1160 dependency not found: libstk_alps_lib.a")
endif()

add_executable(ai-cubpet
    src/main.c
)

target_link_libraries(ai-cubpet PRIVATE
    ${MISC_IO_TARGET}
    ${PM_TARGET}
    ${WIFI_TARGET}
    ${NFC_TARGET}
    -Wl,--whole-archive
    ${MOTOR_TARGET}
    -Wl,--no-whole-archive
    ${LIGHT_SENSOR_TARGET}
    Threads::Threads
    dl
    rt
    m
    stdc++
)

target_link_libraries(ai-cubpet PRIVATE ${STK_ALPS_LIB})

if(LIBGPIOD_FOUND)
    target_include_directories(ai-cubpet PRIVATE ${LIBGPIOD_INCLUDE_DIRS})
    target_link_libraries(ai-cubpet PRIVATE ${LIBGPIOD_LIBRARIES})
    target_compile_options(ai-cubpet PRIVATE ${LIBGPIOD_CFLAGS_OTHER})
else()
    target_link_libraries(ai-cubpet PRIVATE ${LIBGPIOD_LIBRARY})
endif()

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(ai-cubpet PRIVATE
        -Wall
        -Wextra
    )
    target_link_options(ai-cubpet PRIVATE
        -no-pie
    )
endif()

install(TARGETS ai-cubpet
    RUNTIME DESTINATION bin
)
