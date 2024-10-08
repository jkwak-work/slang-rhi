cmake_minimum_required(VERSION 3.20)

project(slang-rhi)

include(CMakeDependentOption)

# Add the cmake directory to the module path.
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Check if this project is the master cmake project (i.e. not included via add_subdirectory).
set(SLANG_RHI_MASTER_PROJECT OFF)
if (${CMAKE_CURRENT_SOURCE_DIR} STREQUAL ${CMAKE_SOURCE_DIR})
  set(SLANG_RHI_MASTER_PROJECT ON)
endif()

# Configuration options
option(SLANG_RHI_BUILD_SHARED "Build shared library" OFF)
option(SLANG_RHI_BUILD_TESTS "Build tests" ON)

# Determine available backends
set(SLANG_RHI_HAS_D3D11 OFF)
set(SLANG_RHI_HAS_D3D12 OFF)
set(SLANG_RHI_HAS_VULKAN OFF)
set(SLANG_RHI_HAS_METAL OFF)
set(SLANG_RHI_HAS_CUDA OFF)
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(SLANG_RHI_HAS_D3D11 ON)
    set(SLANG_RHI_HAS_D3D12 ON)
    set(SLANG_RHI_HAS_VULKAN ON)
    set(SLANG_RHI_HAS_CUDA ON)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(SLANG_RHI_HAS_VULKAN ON)
    set(SLANG_RHI_HAS_CUDA ON)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(SLANG_RHI_HAS_VULKAN ON)
    set(SLANG_RHI_HAS_METAL ON)
endif()

# Backend options
cmake_dependent_option(SLANG_RHI_ENABLE_D3D11 "Enable D3D11 backend" ON "SLANG_RHI_HAS_D3D11" OFF)
cmake_dependent_option(SLANG_RHI_ENABLE_D3D12 "Enable D3D12 backend" ON "SLANG_RHI_HAS_D3D12" OFF)
cmake_dependent_option(SLANG_RHI_ENABLE_VULKAN "Enable Vulkan backend" ON "SLANG_RHI_HAS_VULKAN" OFF)
cmake_dependent_option(SLANG_RHI_ENABLE_METAL "Enable Metal backend" ON "SLANG_RHI_HAS_METAL" OFF)
cmake_dependent_option(SLANG_RHI_ENABLE_CUDA "Enable CUDA backend" ON "SLANG_RHI_HAS_CUDA" OFF)

# If this is the master project, fetch binary dependencies.
if(SLANG_RHI_MASTER_PROJECT)
    include(FetchPackage)
    # Fetch slang
    set(SLANG_VERSION "2024.10")
    set(SLANG_URL "https://github.com/shader-slang/slang/releases/download/v${SLANG_VERSION}/slang-${SLANG_VERSION}")
    if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
        set(SLANG_URL "${SLANG_URL}-windows-x86_64.zip")
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        set(SLANG_URL "${SLANG_URL}-linux-x86_64-glibc-2.17.tar.gz")
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
        if(CMAKE_APPLE_SILICON_PROCESSOR MATCHES "x86_64")
            set(SLANG_URL "${SLANG_URL}-macos-x86_64.zip")
        else()
            set(SLANG_URL "${SLANG_URL}-macos-aarch64.zip")
        endif()
    endif()
    FetchPackage(slang URL ${SLANG_URL})
    set(SLANG_RHI_SLANG_INCLUDE_DIR ${slang_SOURCE_DIR}/include)
    set(SLANG_RHI_SLANG_BINARY_DIR ${slang_SOURCE_DIR})

    if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
        # Fetch nvapi
        # set(NVAPI_VERSION "4ba3384657149d63aa193f5a34e20efe1e42bf31")
        # FetchPackage(nvapi URL "https://github.com/NVIDIA/nvapi/archive/${NVAPI_VERSION}.zip")
        # set(NVAPI_ROOT_DIR ${nvapi_SOURCE_DIR})

        # Fetch dxc
        # set(DXC_VERSION "1.8.2407")
        # set(DXC_TAG "2024_07_31")
        # FetchPackage(dxc URL "https://github.com/microsoft/DirectXShaderCompiler/releases/download/v${DXC_VERSION}/dxc_${DXC_TAG}.zip")

        # Fetch Agility SDK
        # set(AGILITY_SDK_VERSION "1.611.2")
        # FetchPackage(agility_sdk URL "https://www.nuget.org/api/v2/package/Microsoft.Direct3D.D3D12/${AGILITY_SDK_VERSION}")
    endif()
endif()

set(SLANG_RHI_SLANG_INCLUDE_DIR ${SLANG_RHI_SLANG_INCLUDE_DIR} CACHE STRING "Slang include directory")
set(SLANG_RHI_SLANG_BINARY_DIR ${SLANG_RHI_SLANG_BINARY_DIR} CACHE STRING "Slang binary directory")
# Use the variables from the cache.
unset(SLANG_RHI_SLANG_INCLUDE_DIR)
unset(SLANG_RHI_SLANG_BINARY_DIR)

if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-assume -Wno-switch")
endif()

if(CMAKE_CONFIGURATION_TYPES)
    set(SLANG_RHI_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIG>)
else()
    set(SLANG_RHI_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endif()

if(SLANG_RHI_BUILD_SHARED)
    add_library(slang-rhi SHARED)
else()
    add_library(slang-rhi STATIC)
endif()

# target_compile_options(slang-rhi PRIVATE $<$<CXX_COMPILER_ID:AppleClang>:-Wall>) # -Wextra -Wpedantic -Wno-unused-parameter -Wno-missing-field-initializer

if(SLANG_RHI_MASTER_PROJECT)
    add_library(slang SHARED IMPORTED GLOBAL)
    if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
        set_target_properties(slang PROPERTIES
            INTERFACE_INCLUDE_DIRECTORIES ${SLANG_RHI_SLANG_INCLUDE_DIR}
            IMPORTED_IMPLIB ${SLANG_RHI_SLANG_BINARY_DIR}/lib/slang.lib
            IMPORTED_LOCATION ${SLANG_RHI_SLANG_BINARY_DIR}/bin/slang.dll
        )
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        set_target_properties(slang PROPERTIES
            INTERFACE_INCLUDE_DIRECTORIES ${SLANG_RHI_SLANG_INCLUDE_DIR}
            IMPORTED_LOCATION ${SLANG_RHI_SLANG_BINARY_DIR}/lib/libslang.so
        )
    elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
        set_target_properties(slang PROPERTIES
            INTERFACE_INCLUDE_DIRECTORIES ${SLANG_RHI_SLANG_INCLUDE_DIR}
            IMPORTED_LOCATION ${SLANG_RHI_SLANG_BINARY_DIR}/lib/libslang.dylib
        )
    endif()

    target_link_libraries(slang-rhi PUBLIC slang)

    if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
        add_custom_command(
            OUTPUT ${SLANG_RHI_OUTPUT_DIRECTORY}/slang.dll
            DEPENDS ${SLANG_RHI_SLANG_BINARY_DIR}/bin/slang.dll
            COMMAND ${CMAKE_COMMAND} -E copy ${SLANG_RHI_SLANG_BINARY_DIR}/bin/slang.dll ${SLANG_RHI_OUTPUT_DIRECTORY}/
            COMMAND ${CMAKE_COMMAND} -E copy ${SLANG_RHI_SLANG_BINARY_DIR}/bin/slang-glslang.dll ${SLANG_RHI_OUTPUT_DIRECTORY}/
            COMMAND ${CMAKE_COMMAND} -E copy ${SLANG_RHI_SLANG_BINARY_DIR}/bin/slang-llvm.dll ${SLANG_RHI_OUTPUT_DIRECTORY}/
            COMMAND ${CMAKE_COMMAND} -E copy ${SLANG_RHI_SLANG_BINARY_DIR}/bin/slang-rt.dll ${SLANG_RHI_OUTPUT_DIRECTORY}/
            COMMENT "Copying DLLs"
        )
        add_custom_target(slang-rhi-copy-binaries ALL DEPENDS ${SLANG_RHI_OUTPUT_DIRECTORY}/slang.dll)
        add_dependencies(slang slang-rhi-copy-binaries)
    endif()
else()
    target_include_directories(slang-rhi PUBLIC ${SLANG_RHI_SLANG_INCLUDE_DIR})
endif()

file(GLOB RHI_SOURCES
    src/*.cpp
    src/core/*.cpp
    src/cpu/*.cpp
    src/debug-layer/*.cpp
)
target_sources(slang-rhi PRIVATE ${RHI_SOURCES})

if(APPLE)
    file(GLOB OBJC_SOURCES src/*.mm)
    target_sources(slang-rhi PRIVATE ${OBJC_SOURCES})
endif()
if(SLANG_RHI_ENABLE_D3D11 OR SLANG_RHI_HAS_D3D12)
    file(GLOB D3D_SOURCES src/d3d/*.cpp src/nvapi/*.cpp)
    target_sources(slang-rhi PRIVATE ${D3D_SOURCES})
endif()
if(SLANG_RHI_ENABLE_D3D11)
        file(GLOB D3D11_SOURCES src/d3d11/*.cpp)
    target_sources(slang-rhi PRIVATE ${D3D11_SOURCES})
endif()
if(SLANG_RHI_ENABLE_D3D12)
    file(GLOB D3D12_SOURCES src/d3d12/*.cpp)
    target_sources(slang-rhi PRIVATE ${D3D12_SOURCES})
endif()
if(SLANG_RHI_ENABLE_VULKAN)
    file(GLOB VULKAN_SOURCES src/vulkan/*.cpp)
    target_sources(slang-rhi PRIVATE ${VULKAN_SOURCES})

    add_library(slang-rhi-vulkan-headers INTERFACE)
    target_include_directories(slang-rhi-vulkan-headers INTERFACE external/vulkan-headers/include)
    target_link_libraries(slang-rhi PRIVATE slang-rhi-vulkan-headers)
endif()
if(SLANG_RHI_ENABLE_METAL)
    file(GLOB METAL_SOURCES src/metal/*.cpp src/metal/*.mm)
    target_sources(slang-rhi PRIVATE ${METAL_SOURCES})

    add_library(slang-rhi-metal-cpp INTERFACE)
    target_include_directories(slang-rhi-metal-cpp INTERFACE external/metal-cpp)
    target_link_libraries(slang-rhi-metal-cpp INTERFACE "-framework Foundation" "-framework Metal" "-framework QuartzCore")
    target_link_libraries(slang-rhi PRIVATE slang-rhi-metal-cpp)
endif()
if(SLANG_RHI_ENABLE_CUDA)
    file(GLOB CUDA_SOURCES src/cuda/*.cpp)
    target_sources(slang-rhi PRIVATE ${CUDA_SOURCES})
endif()

target_include_directories(slang-rhi PUBLIC include)
target_include_directories(slang-rhi PRIVATE src)
target_compile_definitions(slang-rhi
    PRIVATE
        SLANG_RHI_ENABLE_D3D11=$<BOOL:${SLANG_RHI_ENABLE_D3D11}>
        SLANG_RHI_ENABLE_D3D12=$<BOOL:${SLANG_RHI_ENABLE_D3D12}>
        SLANG_RHI_ENABLE_VULKAN=$<BOOL:${SLANG_RHI_ENABLE_VULKAN}>
        SLANG_RHI_ENABLE_METAL=$<BOOL:${SLANG_RHI_ENABLE_METAL}>
        SLANG_RHI_ENABLE_CUDA=$<BOOL:${SLANG_RHI_ENABLE_CUDA}>
        $<$<PLATFORM_ID:Windows>:NOMINMAX>  # do not define min/max macros
        $<$<PLATFORM_ID:Windows>:UNICODE>   # force character map to unicode
)
target_compile_features(slang-rhi PRIVATE cxx_std_17)
set_target_properties(slang-rhi PROPERTIES POSITION_INDEPENDENT_CODE ON)

if(SLANG_RHI_BUILD_TESTS)
    add_library(doctest INTERFACE)
    target_include_directories(doctest INTERFACE external/doctest)

    add_library(stb INTERFACE)
    target_include_directories(stb INTERFACE external/stb)

    file(GLOB TEST_SOURCES
        tests/*.cpp
    )
    add_executable(slang-rhi-tests)
    target_sources(slang-rhi-tests PRIVATE ${TEST_SOURCES})
    target_compile_definitions(slang-rhi-tests
        PRIVATE
            $<$<PLATFORM_ID:Windows>:NOMINMAX>  # do not define min/max macros
            $<$<PLATFORM_ID:Windows>:UNICODE>   # force character map to unicode
    )
    target_compile_features(slang-rhi-tests PRIVATE cxx_std_17)
    target_include_directories(slang-rhi-tests PRIVATE tests)
    target_link_libraries(slang-rhi-tests PRIVATE doctest stb slang slang-rhi)
endif()
