cmake_minimum_required(VERSION 3.28)

project(lemon_cpp VERSION 9.1.0)

# ============================================================
# Electron source paths (used by both runtime and installers)
# ============================================================
get_filename_component(ELECTRON_APP_SOURCE_DIR "${CMAKE_SOURCE_DIR}/../../src/app" ABSOLUTE)
set(ELECTRON_APP_BUILD_DIR "${ELECTRON_APP_SOURCE_DIR}/dist-app")
if(WIN32)
    set(ELECTRON_APP_UNPACKED_DIR "${ELECTRON_APP_BUILD_DIR}/win-unpacked")
    set(ELECTRON_EXE_NAME "Lemonade.exe")
elseif(APPLE)
    set(ELECTRON_APP_UNPACKED_DIR "${ELECTRON_APP_BUILD_DIR}/mac")
    set(ELECTRON_EXE_NAME "Lemonade.app")
else()
    set(ELECTRON_APP_UNPACKED_DIR "${ELECTRON_APP_BUILD_DIR}/linux-unpacked")
    # Electron builder uses lowercase product name on Linux
    set(ELECTRON_EXE_NAME "lemonade")
endif()

# C++ Standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)

# ============================================================
# Common dependency declarations
# ============================================================

find_package(nlohmann_json REQUIRED)
find_package(CLI11 REQUIRED)
find_package(zstd REQUIRED)
find_package(httplib REQUIRED)
find_package(CURL REQUIRED)

message("curl libs: ${CURL_LIBRARIES}")
message("curl dirs: ${CURL_INCLUDE_DIRS}")

# ============================================================
# Application: lemonade-router
# ============================================================

set(EXECUTABLE_NAME "lemonade-router")


# ============================================================
# Compilation configurations
# ============================================================

# Common compiler definitions
# Enable httplib thread pool with 8 threads
add_compile_definitions(CPPHTTPLIB_THREAD_POOL_COUNT=8)

# ============================================================
# Compilation
# ============================================================

# Generate version header from template
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/include/lemon/version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/include/lemon/version.h
    @ONLY
)

# Include directories
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}
)

# Source files
set(SOURCES
    server/main.cpp
    server/server.cpp
    server/router.cpp
    server/cli_parser.cpp
    server/model_manager.cpp
    server/wrapped_server.cpp
    server/streaming_proxy.cpp
    server/system_info.cpp
    server/utils/http_client.cpp
    server/utils/json_utils.cpp
    server/utils/process_manager.cpp
    server/utils/path_utils.cpp
    server/utils/wmi_helper.cpp
    server/backends/llamacpp_server.cpp
    server/backends/fastflowlm_server.cpp
    server/backends/ryzenaiserver.cpp
    server/backends/whisper_server.cpp
)

# Create executable
add_executable(${EXECUTABLE_NAME} ${SOURCES})



# ============================================================
# Linking
# ============================================================

# Common linking
target_link_libraries(${EXECUTABLE_NAME} PRIVATE
    httplib::httplib
    nlohmann_json::nlohmann_json
    CLI11::CLI11
    CURL::libcurl
    ${CURL_LIBRARIES}
)

# ============================================================
# Resources
# ============================================================

# Copy resources to the build directory during configuration
file(COPY ${CMAKE_SOURCE_DIR}/../../src/lemonade/tools/server/static
     DESTINATION ${CMAKE_BINARY_DIR}/resources)
file(COPY ${CMAKE_SOURCE_DIR}/../../src/lemonade_server/server_models.json
     DESTINATION ${CMAKE_BINARY_DIR}/resources)
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/resources/backend_versions.json
     DESTINATION ${CMAKE_BINARY_DIR}/resources)

# Copy resources to the runtime directoriy after build
add_custom_command(TARGET ${EXECUTABLE_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_BINARY_DIR}/resources
        $<TARGET_FILE_DIR:${EXECUTABLE_NAME}>/resources
    COMMENT "Copying resources to output directory"
)

# ============================================================
# Electron App Integration (Cross-Platform)
# ============================================================

# Find Node.js and npm for building the Electron app
find_program(NODE_EXECUTABLE node)
find_program(NPM_EXECUTABLE npm)

# Custom target to build the Electron app
if(NODE_EXECUTABLE AND NPM_EXECUTABLE)
    if(WIN32)
        set(ELECTRON_BUILD_TARGET "build:win")
    elseif(APPLE)
        set(ELECTRON_BUILD_TARGET "build:mac")
    else()
        set(ELECTRON_BUILD_TARGET "build:linux")
    endif()
    
    add_custom_target(electron-app
        COMMAND ${CMAKE_COMMAND} -E echo "Building Electron app for current platform..."
        COMMAND ${CMAKE_COMMAND} -E chdir "${ELECTRON_APP_SOURCE_DIR}" ${NPM_EXECUTABLE} install
        COMMAND ${CMAKE_COMMAND} -E chdir "${ELECTRON_APP_SOURCE_DIR}" ${NPM_EXECUTABLE} run ${ELECTRON_BUILD_TARGET}
        COMMENT "Building Electron app with npm"
        WORKING_DIRECTORY "${ELECTRON_APP_SOURCE_DIR}"
        VERBATIM
    )
    
    message(STATUS "Node.js found: ${NODE_EXECUTABLE}")
    message(STATUS "npm found: ${NPM_EXECUTABLE}")
    message(STATUS "Electron app can be built with: cmake --build . --target electron-app")
else()
    message(STATUS "Node.js or npm not found - Electron app build target disabled")
    message(STATUS "Install Node.js to enable: https://nodejs.org/")
endif()

# Custom command to copy Electron app files after building the C++ server
add_custom_command(TARGET ${EXECUTABLE_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E echo "Checking for Electron app at ${ELECTRON_APP_UNPACKED_DIR}..."
    COMMAND ${CMAKE_COMMAND}
        -DELECTRON_APP_UNPACKED_DIR=${ELECTRON_APP_UNPACKED_DIR}
        -DTARGET_DIR=$<TARGET_FILE_DIR:${EXECUTABLE_NAME}>
        -DELECTRON_EXE_NAME=${ELECTRON_EXE_NAME}
        -P ${CMAKE_CURRENT_SOURCE_DIR}/CopyElectronApp.cmake
    COMMENT "Copying Electron app if available"
    VERBATIM
)

# Add tray application subdirectory
add_subdirectory(tray)


# ============================================================
# CPack Configuration for .deb Package (Linux only)
# ============================================================
if(UNIX AND NOT APPLE)
    set(CPACK_GENERATOR "DEB")
    set(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")
    set(CPACK_PACKAGE_VENDOR "Lemonade")
    set(CPACK_PACKAGE_CONTACT "lemonade@amd.com")
    set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Lemonade Local LLM Server")
    set(CPACK_PACKAGE_DESCRIPTION "A lightweight, high-performance local LLM server with support for multiple backends including llama.cpp, FastFlowLM, and RyzenAI.")
    
    # Debian-specific settings
    set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Lemonade Team <lemonade@amd.com>")
    set(CPACK_DEBIAN_PACKAGE_SECTION "utils")
    set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
    set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "amd64")
    
    # Installation paths - use ~/.local for user install (no sudo needed)
    # Set via environment: cmake -DCPACK_PACKAGING_INSTALL_PREFIX=$HOME/.local ..
    if(NOT DEFINED CPACK_PACKAGING_INSTALL_PREFIX)
        set(CPACK_PACKAGING_INSTALL_PREFIX "/usr/local")
    endif()
    
    # Only install our executables (not curl/development files)
    # Note: lemonade-server installation is handled in tray/CMakeLists.txt
    install(TARGETS ${EXECUTABLE_NAME}
        RUNTIME DESTINATION bin
    )
    
    # Install resources
    install(DIRECTORY ${CMAKE_BINARY_DIR}/resources/
        DESTINATION share/lemonade-server/resources
    )
    
    # Check if Electron app is available for full package
    option(INCLUDE_ELECTRON_APP "Include Electron app in deb package" OFF)
    
    if(INCLUDE_ELECTRON_APP AND EXISTS "${ELECTRON_APP_UNPACKED_DIR}/${ELECTRON_EXE_NAME}")
        message(STATUS "Electron app found at ${ELECTRON_APP_UNPACKED_DIR}")
        message(STATUS "Building full deb package with Electron app")
        
        # Full package name
        set(CPACK_PACKAGE_NAME "lemonade")
        set(CPACK_DEBIAN_PACKAGE_DEPENDS "libcurl4, libssl3, libz1, unzip, libgtk-3-0, libnotify4, libnss3, libxss1, libxtst6, xdg-utils, libatspi2.0-0, libsecret-1-0")
        set(CPACK_DEBIAN_FILE_NAME "${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}.deb")
        
        # Install Electron app
        install(DIRECTORY "${ELECTRON_APP_UNPACKED_DIR}/"
            DESTINATION share/lemonade-server/app
            USE_SOURCE_PERMISSIONS
            PATTERN "*.so" PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE
        )
        
        # Post-install script for full package
        # Note: CPack requires the script to be named 'postinst' exactly
        # We copy postinst-full to a temporary postinst file for CPack
        configure_file("${CMAKE_CURRENT_SOURCE_DIR}/postinst-full" 
                       "${CMAKE_CURRENT_BINARY_DIR}/postinst" COPYONLY)
        set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_BINARY_DIR}/postinst")
    else()
        # Minimal package (server only)
        set(CPACK_PACKAGE_NAME "lemonade-server-minimal")
        set(CPACK_DEBIAN_PACKAGE_DEPENDS "libcurl4, libssl3, libz1, unzip")
        set(CPACK_DEBIAN_FILE_NAME "${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}.deb")
        
        # Post-install script for minimal package
        set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_SOURCE_DIR}/postinst")
        
        if(INCLUDE_ELECTRON_APP)
            message(WARNING "INCLUDE_ELECTRON_APP is ON but Electron app not found at ${ELECTRON_APP_UNPACKED_DIR}")
            message(WARNING "Building minimal deb package instead")
        endif()
    endif()
    
    include(CPack)
endif()
