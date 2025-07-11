cmake_minimum_required(VERSION 3.14)
set(XBE_TITLE xemu_menu)
set(XBOX_ISO_DIR ${CMAKE_CURRENT_BINARY_DIR}/xiso)
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

project(xemu_menu C CXX ASM)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()



set(CMAKE_ASM_FLAGS_DEBUG "${CMAKE_ASM_FLAGS_DEBUG} -g -gdwarf-4")
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -gdwarf-4")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -gdwarf-4")
set(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG} -debug")

set(CMAKE_ASM_FLAGS_RELEASE "${CMAKE_ASM_FLAGS_RELEASE} -O2")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O2")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O2")

include(FetchContent)
include(FindPkgConfig)
pkg_search_module(SDL2 REQUIRED sdl2)

add_definitions(-DXBOX -DNXDK -DSDL_DISABLE_JOYSTICK_INIT_DELAY -DFTP_CUSTOM_ROOT_PATH -DMEMP_NUM_NETBUF=6 -DMEMP_NUM_NETCONN=6)

add_executable(xemu_menu
    main.c
    menu_main.c
    menu_system_info.c
    menu_eeprom.c
    dvd_autolaunch.c
    network.c
    text.c
    renderer.c
    lib/ftpd/ftp_file.c lib/ftpd/ftp_server.c lib/ftpd/ftp.c
)

target_compile_options(xemu_menu PRIVATE -Wall -Wextra)

target_include_directories(xemu_menu PRIVATE lib)

# Bring in the DVD drive automount support
target_link_libraries(xemu_menu PUBLIC ${NXDK_DIR}/lib/libnxdk_automount_d.lib)
target_link_options(xemu_menu PRIVATE "-include:_automount_d_drive")

# Bring in networking support
target_link_libraries(xemu_menu PUBLIC ${NXDK_DIR}/lib/libnxdk_net.lib)
target_include_directories(xemu_menu PRIVATE ${NXDK_DIR}/lib/net/lwip/src/include ${NXDK_DIR}/lib/net/nforceif/include ${NXDK_DIR}/lib/net/nvnetdrv)

# Bring in pbkit support
target_link_libraries(xemu_menu PUBLIC ${NXDK_DIR}/lib/libpbkit.lib)

# Bring in SDL2 support
target_include_directories(xemu_menu PRIVATE ${SDL2_INCLUDE_DIRS})
target_link_libraries(xemu_menu PUBLIC ${SDL2_LIBRARIES})
target_link_directories(xemu_menu PUBLIC ${SDL2_LIBRARY_DIRS})

#Post-build commands
file(MAKE_DIRECTORY ${XBOX_ISO_DIR})

#Post-build exe to xbe conversion
add_custom_target(cxbe_convert ALL
    COMMENT "CXBE Conversion: [EXE -> XBE]"
    VERBATIM COMMAND "${CMAKE_COMMAND}" -E env ${NXDK_DIR}/tools/cxbe/cxbe -OUT:${CMAKE_CURRENT_BINARY_DIR}/default.xbe -TITLE:${XBE_TITLE} ${CMAKE_CURRENT_BINARY_DIR}/${XBE_TITLE}.exe > NUL 2>&1
)
add_dependencies(cxbe_convert xemu_menu)

#Post-build xbe to xiso conversion
add_custom_target(xbe_iso ALL
    COMMENT "XISO Conversion: [XBE -> XISO]"
    COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_CURRENT_BINARY_DIR}/default.xbe" "${XBOX_ISO_DIR}/default.xbe"
    WORKING_DIRECTORY ${XBOX_ISO_DIR}
    VERBATIM COMMAND "${CMAKE_COMMAND}" -E env ${NXDK_DIR}/tools/extract-xiso/build/extract-xiso -c ${XBOX_ISO_DIR} ${CMAKE_CURRENT_BINARY_DIR}/${XBE_TITLE}.iso
)
add_dependencies(xbe_iso cxbe_convert)

set_target_properties(cxbe_convert PROPERTIES OUTPUT_QUIET ON)
set_target_properties(xbe_iso PROPERTIES OUTPUT_QUIET ON)
