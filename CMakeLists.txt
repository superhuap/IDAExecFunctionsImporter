cmake_minimum_required(VERSION 3.20)

project(IDAExecFunctions64 CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(IDA_SDK_PATH "${CMAKE_CURRENT_SOURCE_DIR}/IDA-SDK/src" CACHE PATH "Path to the 'src' directory of the IDA SDK")

if(NOT EXISTS "${IDA_SDK_PATH}/include/pro.h")
    message(FATAL_ERROR "No IDA SDK at '${IDA_SDK_PATH}'. Pass -DIDA_SDK_PATH=<ida-sdk>/src.")
endif()

# The second entry of each pair is the directory layout used by SDKs older than 9.3
if(WIN32)
    set(IDA_LIBRARY_CANDIDATES
        "${IDA_SDK_PATH}/lib/x64_win_64/ida.lib"
        "${IDA_SDK_PATH}/lib/x64_win_vc_64/ida.lib")
elseif(APPLE)
    # macOS SDKs ship import stubs rather than a real dylib: 9.5+ uses text-based
    # .tbd files, while 9.2-9.4 ship Mach-O stubs named libida.dylib. The 9.5
    # stubs are universal (x86_64 + arm64), so either architecture's copy links.
    set(IDA_MAC_ARCH "arm64")
    if(CMAKE_OSX_ARCHITECTURES)
        if(NOT CMAKE_OSX_ARCHITECTURES MATCHES "arm64")
            set(IDA_MAC_ARCH "x64")
        endif()
    elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|AMD64|amd64)$")
        set(IDA_MAC_ARCH "x64")
    endif()

    set(IDA_LIBRARY_CANDIDATES
        # 9.5+ ships text-based stubs; 9.2-9.4 ship Mach-O stubs named .dylib.
        "${IDA_SDK_PATH}/lib/${IDA_MAC_ARCH}_mac_64/libida.tbd"
        "${IDA_SDK_PATH}/lib/${IDA_MAC_ARCH}_mac_64/libida.dylib"
        # 9.2 used the *_mac_clang_* directory layout.
        "${IDA_SDK_PATH}/lib/${IDA_MAC_ARCH}_mac_clang_64/libida.dylib")
else()
    set(IDA_LIBRARY_CANDIDATES
        "${IDA_SDK_PATH}/lib/x64_linux_64/libida.so"
        "${IDA_SDK_PATH}/lib/x64_linux_gcc_64/libida.so")
endif()

foreach(Candidate IN LISTS IDA_LIBRARY_CANDIDATES)
    if(EXISTS "${Candidate}")
        set(IDA_LIBRARY "${Candidate}")
        break()
    endif()
endforeach()

if(NOT IDA_LIBRARY)
    message(FATAL_ERROR "No IDA library found under '${IDA_SDK_PATH}/lib'.")
endif()

add_library(IDAExecFunctions64 SHARED
    IDAExecFunctions64/FNameConstantNamer.cpp
    IDAExecFunctions64/Format/Parser.cpp
    IDAExecFunctions64/Import/MappingsImporter.cpp
    IDAExecFunctions64/Import/StructBuilder.cpp
    IDAExecFunctions64/Main.cpp
    IDAExecFunctions64/StaticClassNamer.cpp
    IDAExecFunctions64/Utility.cpp
)

target_include_directories(IDAExecFunctions64 PRIVATE
    "${IDA_SDK_PATH}/include"
    "${CMAKE_CURRENT_SOURCE_DIR}/IDAExecFunctions64"
)

target_compile_definitions(IDAExecFunctions64 PRIVATE
    DONT_DEPRECATE
    USE_STANDARD_FILE_FUNCTIONS
)

target_link_libraries(IDAExecFunctions64 PRIVATE "${IDA_LIBRARY}")

set_target_properties(IDAExecFunctions64 PROPERTIES
    PREFIX ""
    POSITION_INDEPENDENT_CODE ON
)

if(MSVC)
    target_compile_definitions(IDAExecFunctions64 PRIVATE _SILENCE_ALL_CXX20_DEPRECATION_WARNINGS)
    target_compile_options(IDAExecFunctions64 PRIVATE /EHsc /utf-8)
else()
    target_compile_options(IDAExecFunctions64 PRIVATE -fno-strict-aliasing -fvisibility=hidden)
    target_link_libraries(IDAExecFunctions64 PRIVATE ${CMAKE_DL_LIBS})
endif()
