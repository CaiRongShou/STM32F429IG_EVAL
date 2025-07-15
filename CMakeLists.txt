cmake_minimum_required(VERSION 3.20) # 最低版本 随便

cmake_policy(SET CMP0123 NEW) 

set(CMAKE_SYSTEM_NAME Generic) #单片机上裸机，不带系统

set(CPU_CORE cortex-m4) #自定义变量名 CPU核
set(CMAKE_SYSTEM_PROCESSOR ${CPU_CORE})

set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_LIST_DIR}/toolchain.cmake) # 工具链

project(output C CXX ASM) #工程名，支持的语法 C ASM
set(CMAKE_C_STANDARD 99) # C语言标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
#---------------------------上述是固定模版---------------------------

# 全局宏定义
ADD_DEFINITIONS(
	-DUSE_HAL_DRIVER 
	-DSTM32F429xx
    # $<$<CONFIG:Debug>:DEBUG>
)

# 头文件
include_directories(
    ${CMAKE_CURRENT_LIST_DIR}/Core/Inc
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Inc
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Inc/Legacy
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx/Include
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/CMSIS/Include
)

# 源文件
file(GLOB DIRVES_FILE
    ${CMAKE_CURRENT_LIST_DIR}/Core/Src/main.c
    ${CMAKE_CURRENT_LIST_DIR}/Core/Src/gpio.c
    ${CMAKE_CURRENT_LIST_DIR}/Core/Src/stm32f4xx_it.c
    ${CMAKE_CURRENT_LIST_DIR}/Core/Src/stm32f4xx_hal_msp.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_rcc.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_rcc_ex.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_flash.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_flash_ex.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_flash_ramfunc.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_gpio.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_dma_ex.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_dma.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_pwr.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_pwr_ex.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_cortex.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal.c
    ${CMAKE_CURRENT_LIST_DIR}/Drivers/STM32F4xx_HAL_Driver/Src/stm32f4xx_hal_exti.c
    ${CMAKE_CURRENT_LIST_DIR}/Core/Src/system_stm32f4xx.c
    ${CMAKE_CURRENT_LIST_DIR}/Core/Src/sysmem.c
)
# 源文件
if(USER_ARMGCC) #  armgcc
file(GLOB SYSTEM_FILE
    ${CMAKE_CURRENT_LIST_DIR}/Core/Src/syscalls.c
)
    set(SOURCE_FILE ${DIRVES_FILE} ${SYSTEM_FILE})
else() #  armclang
    set(SOURCE_FILE ${DIRVES_FILE})

endif()

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/output) # 输出文件路径
set(HEX_FILE ${EXECUTABLE_OUTPUT_PATH}/output.hex) #
set(BIN_FILE ${EXECUTABLE_OUTPUT_PATH}/output.bin)
set(MAP_FILE ${EXECUTABLE_OUTPUT_PATH}/output.map)

if(USER_ARMGCC) #  armgcc
    set(STARTUP_FILE ${CMAKE_CURRENT_LIST_DIR}/startup_stm32f429xx.s) # 启动文件
    set(LINK_FILE ${CMAKE_CURRENT_LIST_DIR}/stm32f429igtx_flash.ld) # 链接文件
    
    # MCU specific flags
    set(TARGET_FLAGS "-mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard ")

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${TARGET_FLAGS}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -Wpedantic -fdata-sections -ffunction-sections")
    if(CMAKE_BUILD_TYPE MATCHES Debug)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g3")
    endif()
    if(CMAKE_BUILD_TYPE MATCHES Release)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Os -g0")
    endif()

    set(CMAKE_ASM_FLAGS "${CMAKE_C_FLAGS} -x assembler-with-cpp -MMD -MP")
    set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -fno-rtti -fno-exceptions -fno-threadsafe-statics")

    set(CMAKE_C_LINK_FLAGS "${TARGET_FLAGS}")
    set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -T ${LINK_FILE}")
    set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} --specs=nano.specs")
    set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,-Map=${CMAKE_PROJECT_NAME}.map -Wl,--gc-sections")
    set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,--start-group -lc -lm -Wl,--end-group")
    set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,--print-memory-usage")

    set(CMAKE_CXX_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,--start-group -lstdc++ -lsupc++ -Wl,--end-group")

    add_executable(${PROJECT_NAME}.elf ${SOURCE_FILE} ${STARTUP_FILE}) # 创建可执行文件
    
    #  设置可执行文件输出路径
    set_target_properties(${PROJECT_NAME}.elf PROPERTIES 
        RUNTIME_OUTPUT_DIRECTORY ${EXECUTABLE_OUTPUT_PATH}
    )

    add_custom_command(
        TARGET ${PROJECT_NAME}.elf # 目标文件
        POST_BUILD #  编译完成之后执行
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}.elf> ${BIN_FILE}
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}.elf> ${HEX_FILE}
        COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${PROJECT_NAME}.elf>
    )   

else() #  armclang
    set(STARTUP_FILE ${CMAKE_CURRENT_LIST_DIR}/startup/ARM/startup_gd32f407_427.s) # 启动文件
    set(LINK_FILE ${CMAKE_CURRENT_LIST_DIR}/startup/linker_scripts/link.sct) # 链接文件

    if(NOT EXISTS ${LINK_FILE})
        message(FATAL_ERROR "Link script not found: ${LINK_FILE}")
    endif()

    set(C_CXX_COMPILE_OPTIONS
        # -xc -std=c11 
        -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard -c
        -fno-rtti -funsigned-char -fshort-enums -fshort-wchar -fno-exceptions
        -O1 -ffunction-sections
        -Wno-packed -Wno-missing-variable-declarations -Wno-missing-prototypes
        -Wno-missing-noreturn -Wno-sign-conversion -Wno-nonportable-include-path
        -Wno-reserved-id-macro -Wno-unused-macros -Wno-documentation-unknown-command
        -Wno-documentation -Wno-license-management -Wno-parentheses-equality
        -Wno-reserved-identifier
        )

    set(ASM_COMPILE_OPTIONS
        --target=arm-arm-none-eabi -mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard -masm=auto -c
        -Wa,armasm,--diag_suppress=A1950W -c
        )
        
    set_source_files_properties(${CMAKE_CURRENT_LIST_DIR}/startup/startup_gd32f407_427.s PROPERTIES LANGUAGE ASM)

    add_executable(${PROJECT_NAME} ${SOURCE_FILE} ${STARTUP_FILE}) # 创建可执行文件


    if(CMAKE_BUILD_TYPE MATCHES Debug)
        add_compile_options(-O0 g3)
    endif()
    if(CMAKE_BUILD_TYPE MATCHES Release)
        add_compile_options(-Os -g0)
    endif()

    target_compile_options(${PROJECT_NAME} PRIVATE
        $<$<COMPILE_LANGUAGE:C>:${C_CXX_COMPILE_OPTIONS}>
        $<$<COMPILE_LANGUAGE:CXX>:${C_CXX_COMPILE_OPTIONS}>
        $<$<COMPILE_LANGUAGE:ASM>:${ASM_COMPILE_OPTIONS}>
    )

    target_link_options(${PROJECT_NAME} PRIVATE
        --cpu=Cortex-M4.fp.sp
        --scatter=${LINK_FILE}
        --summary_stderr
        --map
        --info=totals
        --info=sizes
        --list=${CMAKE_BINARY_DIR}/${PROJECT_NAME}.map
    )


    #  设置可执行文件输出路径
    set_target_properties(${PROJECT_NAME} PROPERTIES 
        RUNTIME_OUTPUT_DIRECTORY ${EXECUTABLE_OUTPUT_PATH}
    )

    add_custom_command(
        TARGET ${PROJECT_NAME}
        POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} --i32combined --output=${HEX_FILE} $<TARGET_FILE:${PROJECT_NAME}>
        COMMAND ${CMAKE_OBJCOPY} --bincombined --output=${BIN_FILE} $<TARGET_FILE:${PROJECT_NAME}>
        COMMAND ${CMAKE_SIZE} --text --info=sizes $<TARGET_FILE:${PROJECT_NAME}>
    )

endif()

