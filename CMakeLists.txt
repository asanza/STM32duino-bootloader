cmake_minimum_required(VERSION 3.17)

# Enable export of compilation database for further analysis
# This creates a compile_comands.json file in the build directory that
# can be used by visual studio and cppcheck for code completion and
# build analysis.
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# set the path to custom modules
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
# set path to the scripts
set(SCRIPTS_PATH "${CMAKE_SOURCE_DIR}/tools/scripts")

if( NOT CMAKE_TOOLCHAIN_FILE )
  message("-- Info: Using the GNU Toolchain" )
  message("-- Info: Set the toolchain file with -DCMAKE_TOOLCHAIN_FILE=/toolchain/file/path")
  set(CMAKE_TOOLCHAIN_FILE ${CMAKE_MODULE_PATH}/arm-gcc-toolchain.cmake)
endif()

#find python, as is needed for some scripts
find_package(Python 3.7 REQUIRED )
set(PYTHON  "${Python_EXECUTABLE}")

if(NOT Python_FOUND )
message(FATAL_ERROR "Python3 not found. You need Python 3 to build this program")
endif()

project(STM32duino-bootloader LANGUAGES C CXX ASM DESCRIPTION "STM32 Arduino Bootloader.")

add_compile_options(
  -mcpu=cortex-m3
  -mthumb
  -mthumb-interwork
  -Os
  -g3
  -ggdb
  -fdiagnostics-color=always
  -ffunction-sections
  -fdata-sections
  -fmerge-all-constants
  -Wall
  -Wextra
  -Wpedantic
  -Wfatal-errors
)

add_link_options(
    -mcpu=cortex-m3
    -mthumb
    -mthumb-interwork
    -nostartfiles
    -Wl,--gc-sections
    -lc
    -lgcc
)

if( CMAKE_BUILD_TYPE STREQUAL "Debug" )
  add_compile_options(-Og)
  message("-- Warning: This is a DEBUG build. Not suitable for release purposes ")
  message("-- Enabling debug symbols ")
else()
  message("-- Info: This is a release build")
  # enable size optimization
  add_compile_options(-Os)
  add_compile_options(-fexpensive-optimizations)
  add_compile_definitions(NDEBUG)
endif()

# include git version extraction
include(${CMAKE_MODULE_PATH}/gitversion.cmake)
get_repository_version()
add_definitions(-DGIT_VERSION=\"${GIT_VERSION}\")
add_definitions(-DGIT_VERSION_MAJOR=${GIT_VERSION_MAJOR})
add_definitions(-DGIT_VERSION_MINOR=${GIT_VERSION_MINOR})
add_definitions(-DGIT_VERSION_PATCH_LEVEL=${GIT_VERSION_PATCH_LEVEL})
add_definitions(-DGIT_COMMIT_HASH=${GIT_COMMIT_HASH})
add_definitions(-DGIT_COMMIT_HASH_HEX=0x${GIT_COMMIT_HASH})

set(SOURCES
    src/usb_lib/usb_regs.c
    src/usb_lib/usb_mem.c
    src/usb_lib/usb_int.c
    src/usb_lib/usb_init.c
    src/usb_lib/usb_core.c
    src/dfu.c
    src/hardware.c
    src/main.c
    src/usb_callbacks.c
    src/usb_descriptor.c
    src/usb.c
    src/stm32_lib/c_only_startup.s
    src/stm32_lib/cortexm3_macro.s
)

add_executable(generic-pc13 ${SOURCES})
target_include_directories(generic-pc13 PRIVATE src src/usb_lib src/stm32_lib)
target_compile_definitions(generic-pc13 PRIVATE TARGET_GENERIC_F103_PC13 )
target_link_options(generic-pc13 PRIVATE -T${CMAKE_CURRENT_LIST_DIR}/src/stm32_lib/c_only_md_high_density.ld)
add_custom_target(generic-pc13-hex ALL
    ${CMAKE_OBJCOPY} -Oihex generic-pc13 generic-pc13.hex
    DEPENDS generic-pc13
)