cmake_minimum_required(VERSION 3.16)

set(CMAKE_TOOLCHAIN_FILE $ENV{IDF_PATH}/tools/cmake/toolchain-esp32s3.cmake CACHE STRING "")

project(ardupilot)

# Include for ESP-IDF build system functions
include($ENV{IDF_PATH}/tools/cmake/idf.cmake)

# Create idf::esp32 and idf::freertos static libraries
idf_build_process(esp32s3
    # try and trim the build; additional components
    # will be included as needed based on dependency tree
    #
    # although esptool_py does not generate static library,
    # processing the component is needed for flashing related
    # targets and file generation
    COMPONENTS freertos
               fatfs
               nvs_flash
               esptool_py
               app_update
               esp_adc
               esp_driver_mcpwm
               driver
               lwip
               esp_wifi
               esp_system
               esp_rom
               esp_timer

    # treat sdkconfig as build product generated by the defaults
    SDKCONFIG ${CMAKE_BINARY_DIR}/sdkconfig
    SDKCONFIG_DEFAULTS ${CMAKE_CURRENT_LIST_DIR}/sdkconfig.defaults
    BUILD_DIR ${CMAKE_BINARY_DIR}
)

set(elf_file ardupilot.elf)

add_executable(${elf_file} main.c)

if(NOT DEFINED ARDUPILOT_CMD)
    set(ARDUPILOT_CMD "none")
endif()

message("WAF_BUILD_TARGET=${WAF_BUILD_TARGET}")

string(REGEX MATCH "^(examples|tool)/" IS_EXAMPLE "${WAF_BUILD_TARGET}")

if (IS_EXAMPLE)
    string(REPLACE "/" ";" A ${WAF_BUILD_TARGET})
    list(GET A 0 EXAMPLE_BASE)
    list(GET A 1 EXAMPLE_NAME)
    message("Building ${EXAMPLE_BASE} ${EXAMPLE_NAME}")
    target_link_libraries(${elf_file} "${ARDUPILOT_LIB}/${EXAMPLE_BASE}/lib${EXAMPLE_NAME}.a")
    target_link_libraries(${elf_file} "${ARDUPILOT_LIB}/libap.a")
ELSEIF(${ARDUPILOT_CMD} STREQUAL "plane")
    message("Building for plane")
    target_link_libraries(${elf_file} "${ARDUPILOT_BIN}/libarduplane.a")
    target_link_libraries(${elf_file} "${ARDUPILOT_LIB}/libArduPlane_libs.a")
ELSEIF(${ARDUPILOT_CMD} STREQUAL "copter")
    message("Building for copter")
    target_link_libraries(${elf_file} "${ARDUPILOT_BIN}/libarducopter.a")
    target_link_libraries(${elf_file} "${ARDUPILOT_LIB}/libArduCopter_libs.a")
ELSEIF(${ARDUPILOT_CMD} STREQUAL "rover")
    message("Building for rover")
    target_link_libraries(${elf_file} "${ARDUPILOT_BIN}/libardurover.a")
    target_link_libraries(${elf_file} "${ARDUPILOT_LIB}/libRover_libs.a")
ELSEIF(${ARDUPILOT_CMD} STREQUAL "sub")
    message("Building for submarine")
    target_link_libraries(${elf_file} "${ARDUPILOT_BIN}/libardusub.a")
    target_link_libraries(${elf_file} "${ARDUPILOT_LIB}/libArduSub_libs.a")
ENDIF()

add_custom_target(showinc ALL
        COMMAND echo -e
        "$<TARGET_PROPERTY:${elf_file},INCLUDE_DIRECTORIES>"
        > includes.list
    VERBATIM
    BYPRODUCTS includes.list
    COMMAND_EXPAND_LISTS
)

#Find files name lib to link
#function(SUBLINK target curdir)
#  FILE(GLOB children RELATIVE ${curdir} ${curdir}/*)
#  FOREACH(child ${children})
#    IF(NOT IS_DIRECTORY ${curdir}/${child})
#      SET(PATH "${curdir}/${child}")
#      message("Linking ${PATH}")
#      target_link_libraries(${target} "${PATH}")
#    ENDIF()
#  ENDFOREACH()
#endfunction()

#IF (DEFINED ARDUPILOT_BIN)
#    SUBLINK(${elf_file} ${ARDUPILOT_BIN})
#ENDIF()
#IF (DEFINED ARDUPILOT_LIB)
#    SUBLINK(${elf_file} ${ARDUPILOT_LIB})
#ENDIF()

# Link the static libraries to the executable
target_link_libraries(${elf_file}
                idf::freertos
                idf::fatfs
                idf::nvs_flash
                idf::spi_flash
                idf::app_update
                idf::esp_adc
                idf::esp_driver_mcpwm
                idf::driver
                idf::lwip
                idf::esp_wifi
                idf::esp_system
                idf::esp_rom
                idf::esp_timer
                )

# Attach additional targets to the executable file for flashing,
# linker script generation, partition_table generation, etc.
idf_build_executable(${elf_file})

# wrap malloc to ensure memory is zeroed
target_link_options(${elf_file} PRIVATE "-Wl,--wrap,malloc")

set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

# Additional targets for measuring RAM use: size, size-components, size-files
#   - Adapted from ${IDF_PATH}/tools/cmake/project.cmake
#
# Reference:
#   - https://docs.espressif.com/projects/esp-idf/en/v5.0/esp32s3/api-guides/performance/size.html#minimizing-binary-size
#
# Usage:
#   cd ./build/esp32s3xxx/esp-idf_build
#   ninja -v -v size
#
set(mapfile "${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.map")
# Add cross-reference table to the map file
target_link_options(${elf_file} PRIVATE "-Wl,--cref")
# Add this symbol as a hint for esp_idf_size to guess the target name
target_link_options(${elf_file} PRIVATE "-Wl,--defsym=IDF_TARGET_${idf_target}=0")
# Enable map file output
target_link_options(${elf_file} PRIVATE "-Wl,--Map=${mapfile}")
# Check if linker supports --no-warn-rwx-segments
execute_process(COMMAND ${CMAKE_LINKER} "--no-warn-rwx-segments" "--version"
    RESULT_VARIABLE result
    OUTPUT_QUIET
    ERROR_QUIET)
if(${result} EQUAL 0)
    # Do not print RWX segment warnings
    target_link_options(${elf_file} PRIVATE "-Wl,--no-warn-rwx-segments")
endif()
if(CONFIG_ESP_ORPHAN_SECTION_WARNING)
    # Print warnings if orphan sections are found
    target_link_options(${elf_file} PRIVATE "-Wl,--orphan-handling=warn")
endif()

idf_build_get_property(idf_path IDF_PATH)
idf_build_get_property(python PYTHON)

set(idf_size ${python} -m esp_idf_size)

# Add size targets, depend on map file, run esp_idf_size
# OUTPUT_JSON is passed for compatibility reasons, SIZE_OUTPUT_FORMAT
# environment variable is recommended and has higher priority
add_custom_target(size
    COMMAND ${CMAKE_COMMAND}
    -D "IDF_SIZE_TOOL=${idf_size}"
    -D "MAP_FILE=${mapfile}"
    -D "OUTPUT_JSON=${OUTPUT_JSON}"
    -P "${idf_path}/tools/cmake/run_size_tool.cmake"
    DEPENDS ${mapfile}
    USES_TERMINAL
    VERBATIM
)

add_custom_target(size-files
    COMMAND ${CMAKE_COMMAND}
    -D "IDF_SIZE_TOOL=${idf_size}"
    -D "IDF_SIZE_MODE=--files"
    -D "MAP_FILE=${mapfile}"
    -D "OUTPUT_JSON=${OUTPUT_JSON}"
    -P "${idf_path}/tools/cmake/run_size_tool.cmake"
    DEPENDS ${mapfile}
    USES_TERMINAL
    VERBATIM
)

add_custom_target(size-components
    COMMAND ${CMAKE_COMMAND}
    -D "IDF_SIZE_TOOL=${idf_size}"
    -D "IDF_SIZE_MODE=--archives"
    -D "MAP_FILE=${mapfile}"
    -D "OUTPUT_JSON=${OUTPUT_JSON}"
    -P "${idf_path}/tools/cmake/run_size_tool.cmake"
    DEPENDS ${mapfile}
    USES_TERMINAL
    VERBATIM
)
