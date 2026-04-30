cmake_minimum_required(VERSION 3.22)

project(firmware)

# firmware generated include paths
set(
        FIRMWARE_INC
        ../../firmware/CMSIS/
        ../../firmware/CMSIS/GD/GD32H73x_75x/Include
        ../../firmware/GD32H73x_75x_standard_peripheral/Include
)

file(
        GLOB FIRMWARE_SRC
        ../../firmware/CMSIS/GD/GD32H73x_75x/Source/GCC/startup_gd32h73x_75x.S
        ../../firmware/CMSIS/GD/GD32H73x_75x/Source/GCC/newlib/syscalls.c
        ../../firmware/GD32H73x_75x_standard_peripheral/Source/*.c
)

add_library(${PROJECT_NAME} OBJECT ${FIRMWARE_SRC})

target_include_directories(
        ${PROJECT_NAME}
        PUBLIC ${FIRMWARE_INC}
)

# Validate that code is compatible with C standard
if((CMAKE_C_STANDARD EQUAL 90) OR (CMAKE_C_STANDARD EQUAL 99))
    message(ERROR "Generated code requires C11 or higher")
endif()