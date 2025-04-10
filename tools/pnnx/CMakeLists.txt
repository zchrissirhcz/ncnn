
if(NOT CMAKE_VERSION VERSION_LESS "3.15")
    # enable CMAKE_MSVC_RUNTIME_LIBRARY
    cmake_policy(SET CMP0091 NEW)
endif()

project(pnnx)
cmake_minimum_required(VERSION 3.12)

if(POLICY CMP0074)
    cmake_policy(SET CMP0074 NEW)
endif()

if(MSVC AND NOT CMAKE_VERSION VERSION_LESS "3.15")
    option(PNNX_BUILD_WITH_STATIC_CRT "Enables use of statically linked CRT for statically linked pnnx" OFF)
    if(PNNX_BUILD_WITH_STATIC_CRT)
        # cmake before version 3.15 not work
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    endif()
endif()

if(POLICY CMP0094)
    cmake_policy(SET CMP0094 NEW)  # FindPython should return the first matching Python
endif()
set(Python_FIND_REGISTRY "LAST")
set(Python_FIND_FRAMEWORK "LAST")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(PNNXPyTorch)

# c++14 is required for using torch headers
set(CMAKE_CXX_STANDARD 14)

# set(CMAKE_BUILD_TYPE debug)
# set(CMAKE_BUILD_TYPE relwithdebinfo)
# set(CMAKE_BUILD_TYPE release)

option(PNNX_COVERAGE "build for coverage" OFF)

# set(Torch_INSTALL_DIR "/home/nihui/osd/pnnx/install" CACHE STRING "")
# set(TorchVision_INSTALL_DIR "/home/nihui/osd/pnnx/install" CACHE STRING "")
# set(protobuf_DIR "/home/nihui/osd/pnnx/install/lib64/cmake/protobuf" CACHE STRING "")

# set(Torch_DIR "${Torch_INSTALL_DIR}/share/cmake/Torch")
set(TorchVision_DIR "${TorchVision_INSTALL_DIR}/share/cmake/TorchVision")

find_package(Python3 COMPONENTS Interpreter Development)

PNNXProbeForPyTorchInstall()
find_package(Torch REQUIRED)

find_package(TorchVision QUIET)

message(STATUS "Torch_VERSION = ${Torch_VERSION}")
message(STATUS "Torch_VERSION_MAJOR = ${Torch_VERSION_MAJOR}")
message(STATUS "Torch_VERSION_MINOR = ${Torch_VERSION_MINOR}")
message(STATUS "Torch_VERSION_PATCH = ${Torch_VERSION_PATCH}")

if(Torch_VERSION VERSION_LESS "1.8")
    message(FATAL_ERROR "pnnx only supports PyTorch >= 1.8")
endif()

if(Torch_VERSION VERSION_GREATER_EQUAL "2.1")
    # c++17 is required for using torch 2.1+ headers
    set(CMAKE_CXX_STANDARD 17)
endif()

# find torchvision library
find_library(TORCHVISION_LIBRARY torchvision PATHS "${TorchVision_INSTALL_DIR}/lib" "${TorchVision_INSTALL_DIR}/lib64")
if(TORCHVISION_LIBRARY)
    message(STATUS "Found TorchVision: ${TORCHVISION_LIBRARY}")
    if(APPLE)
        set(TORCHVISION_LIBRARY "-Wl,-force_load,${TORCHVISION_LIBRARY}")
    elseif(MSVC)
        set(TORCHVISION_LIBRARY "-WHOLEARCHIVE:${TORCHVISION_LIBRARY}")
    else()
        set(TORCHVISION_LIBRARY "-Wl,--whole-archive ${TORCHVISION_LIBRARY} -Wl,--no-whole-archive")
    endif()
    set(TorchVision_FOUND TRUE)
    message(STATUS "Building with TorchVision")
else()
    message(WARNING "static library ${TORCHVISION_LIBRARY} not found.")
    set(TorchVision_FOUND FALSE)
    message(WARNING "Building without TorchVision")
endif()

include_directories(SYSTEM ${TORCH_INCLUDE_DIRS})

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    # test if libtorch and protobuf has the same cxxabi version
    include(CheckCXXSourceCompiles)
    set(CMAKE_REQUIRED_FLAGS "${TORCH_CXX_FLAGS}")
    check_cxx_source_compiles("#include <cxxabi.h>\n#if _GLIBCXX_USE_CXX11_ABI\nint main() { return 0; }\n#endif" PNNX_TORCH_USE_CXX11_ABI)
    unset(CMAKE_REQUIRED_FLAGS)
    check_cxx_source_compiles("#include <cxxabi.h>\n#if _GLIBCXX_USE_CXX11_ABI\nint main() { return 0; }\n#endif" PNNX_COMPILER_USE_CXX11_ABI)
endif()

if((PNNX_TORCH_USE_CXX11_ABI AND PNNX_COMPILER_USE_CXX11_ABI) OR (NOT PNNX_TORCH_USE_CXX11_ABI AND NOT PNNX_COMPILER_USE_CXX11_ABI))
    find_package(protobuf CONFIG)

    if(protobuf_FOUND)
        set(PROTOBUF_FOUND ${protobuf_FOUND})
        set(PROTOBUF_VERSION ${protobuf_VERSION})
    else()
        # fallback to system
        find_package(Protobuf)
        set(PROTOBUF_FOUND ${Protobuf_FOUND})
        set(PROTOBUF_VERSION ${Protobuf_VERSION})
        if(TARGET protobuf::protoc)
            set_target_properties(protobuf::protoc PROPERTIES IMPORTED_LOCATION_RELEASE "${PROTOBUF_PROTOC_EXECUTABLE}")
        endif()
    endif()
endif()

# https://github.com/supertone-inc/onnxruntime-build
set(onnxruntime_INSTALL_DIR "/home/nihui/osd/pnnx/install" CACHE STRING "")
find_library(onnxruntime_LIB NAMES onnxruntime PATHS ${onnxruntime_INSTALL_DIR}/lib64 ${onnxruntime_INSTALL_DIR}/lib)
if(onnxruntime_LIB)
    set(onnxruntime_FOUND TRUE)
    add_library(onnxruntime::onnxruntime STATIC IMPORTED)
    set_target_properties(onnxruntime::onnxruntime PROPERTIES IMPORTED_LOCATION ${onnxruntime_LIB})
    set_target_properties(onnxruntime::onnxruntime PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${onnxruntime_INSTALL_DIR}/include)
else()
    set(onnxruntime_FOUND FALSE)
endif()

option(PNNX_TNN2PNNX "build tnn2pnnx" ON)

add_subdirectory(src)

enable_testing()
add_subdirectory(tests)
