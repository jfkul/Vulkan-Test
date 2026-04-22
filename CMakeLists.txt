cmake_minimum_required(VERSION 3.29)

set(TEMPLATE_NAME "vulkan_test")
project(${TEMPLATE_NAME})

# --- Dependencies via FetchContent (Solves the "Not Found" errors) ---
include(FetchContent)

# tinyobjloader - switching to master branch to avoid tag errors
FetchContent_Declare(
        tinyobjloader
        GIT_REPOSITORY https://github.com/tinyobjloader/tinyobjloader.git
        GIT_TAG        966edceaf8cdca7996c4e9a1c5ced2938de63366
)

# tinygltf
FetchContent_Declare(
        tinygltf
        GIT_REPOSITORY https://github.com/syoyo/tinygltf.git
        GIT_TAG        release
)

# stb (Header-only)
FetchContent_Declare(
        stb
        GIT_REPOSITORY https://github.com/nothings/stb.git
        GIT_TAG        master
)

FetchContent_MakeAvailable(tinyobjloader tinygltf stb)

# --- Standard System Packages ---
find_package(glfw3 REQUIRED)
find_package(glm REQUIRED)
find_package(Vulkan 1.4 REQUIRED)

# KTX can be tricky on Arch; we'll try to find it, but you may need the 'libktx' AUR package
find_package(KTX QUIET)

# --- Configuration Options ---
option(ENABLE_CPP20_MODULE "Enable C++ 20 module support for Vulkan" OFF)

if(ENABLE_CPP20_MODULE)
    set(CMAKE_CXX_SCAN_FOR_MODULES ON)
endif()

# --- Vulkan C++ Module Setup ---
if(ENABLE_CPP20_MODULE)
    add_library(VulkanCppModule)
    add_library(Vulkan::cppm ALIAS VulkanCppModule)

    target_compile_definitions(VulkanCppModule PUBLIC VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1 VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1)
    target_include_directories(VulkanCppModule PUBLIC "${Vulkan_INCLUDE_DIR}")
    target_link_libraries(VulkanCppModule PUBLIC Vulkan::Vulkan)
    set_target_properties(VulkanCppModule PROPERTIES CXX_STANDARD 20)

    target_sources(VulkanCppModule
            PUBLIC
            FILE_SET cxx_modules TYPE CXX_MODULES
            BASE_DIRS "${Vulkan_INCLUDE_DIR}"
            FILES "${Vulkan_INCLUDE_DIR}/vulkan/vulkan.cppm"
    )
else()
    add_library(VulkanCppModule INTERFACE)
    add_library(Vulkan::cppm ALIAS VulkanCppModule)
    target_link_libraries(VulkanCppModule INTERFACE Vulkan::Vulkan)
    target_compile_definitions(VulkanCppModule INTERFACE VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1 VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1)
endif()

# --- Executable Setup ---
add_executable(${TEMPLATE_NAME} main.cpp)

set(TEMPLATE_DIR ${CMAKE_BINARY_DIR}/${TEMPLATE_NAME})
set_target_properties(${TEMPLATE_NAME} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY ${TEMPLATE_DIR}
        CXX_STANDARD 20
        CXX_STANDARD_REQUIRED ON
)

# Link everything
# Note: tinyobjloader creates a target named 'tinyobjloader' automatically
target_link_libraries(${TEMPLATE_NAME} PRIVATE
        Vulkan::cppm
        glfw
        glm::glm
        tinyobjloader
)

# Include directories for header-only libs
target_include_directories(${TEMPLATE_NAME} PRIVATE
        ${stb_SOURCE_DIR}
        ${tinygltf_SOURCE_DIR}
)

# --- Shader Compilation (Slang) ---
find_program(SLANGC_EXECUTABLE slangc HINTS $ENV{VULKAN_SDK}/bin REQUIRED)

set(SHADER_TARGET ${TEMPLATE_NAME}Shader)
set(SHADERS_DIR ${TEMPLATE_DIR}/shaders)
set(SHADER_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/shader.slang")
set(ENTRY_POINTS -entry vertMain -entry fragMain)

add_custom_command(OUTPUT ${SHADERS_DIR} COMMAND ${CMAKE_COMMAND} -E make_directory ${SHADERS_DIR})
add_custom_command(
        OUTPUT  ${SHADERS_DIR}/slang.spv
        COMMAND ${SLANGC_EXECUTABLE} ${SHADER_SOURCES} -target spirv -profile spirv_1_4 -emit-spirv-directly -fvk-use-entrypoint-name ${ENTRY_POINTS} -o ${SHADERS_DIR}/slang.spv
        WORKING_DIRECTORY ${SHADERS_DIR}
        DEPENDS ${SHADERS_DIR} ${SHADER_SOURCES}
        COMMENT "Compiling Slang Shaders"
        VERBATIM
)
add_custom_target(${SHADER_TARGET} DEPENDS ${SHADERS_DIR}/slang.spv)
add_dependencies(${TEMPLATE_NAME} ${SHADER_TARGET})

# --- Assets ---
set(ASSET_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../assets")
if(EXISTS ${ASSET_DIR})
    file(COPY ${ASSET_DIR}/viking_room.obj DESTINATION ${TEMPLATE_DIR}/models)
    file(COPY ${ASSET_DIR}/viking_room.png DESTINATION ${TEMPLATE_DIR}/textures)
endif()

# Global Definitions
target_compile_definitions(${TEMPLATE_NAME} PRIVATE "VULKAN_HPP_HANDLE_ERROR_OUT_OF_DATE_AS_SUCCESS")
if(ENABLE_CPP20_MODULE)
    target_compile_definitions(${TEMPLATE_NAME} PRIVATE USE_CPP20_MODULES=1)
endif()