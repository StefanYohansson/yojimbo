cmake_minimum_required(VERSION 3.20)

# Project definition
project(YojimboProject LANGUAGES C CXX)

# Set C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Define build configurations
set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "Available build configurations" FORCE)

# Common include directories
set(COMMON_INCLUDE_DIRS
        ${CMAKE_SOURCE_DIR}
        ${CMAKE_SOURCE_DIR}/include
        ${CMAKE_SOURCE_DIR}/sodium
        ${CMAKE_SOURCE_DIR}/tlsf
        ${CMAKE_SOURCE_DIR}/netcode
        ${CMAKE_SOURCE_DIR}/reliable
        ${CMAKE_SOURCE_DIR}/serialize
)

# Apply common include directories
include_directories(${COMMON_INCLUDE_DIRS})

# Common compiler options
if(MSVC)
    # Disable RTTI
    add_compile_options(/GR-)
    # Extra warnings and treat warnings as errors
    add_compile_options(/W4 /WX)
    # Fast floating point
    add_compile_options(/fp:fast)
else()
    # Disable RTTI
    add_compile_options(-fno-rtti)
    # Extra warnings and treat warnings as errors
    add_compile_options(-Wall -Wextra -Wfatal-errors)
    # Fast floating point
    add_compile_options(-ffast-math)
endif()

# =====================================
# Define Static Libraries
# =====================================

# sodium-builtin
add_library(sodium-builtin STATIC sodium/dummy.c)

if(WIN32)
    # Include all .c and .h files for Windows
    file(GLOB SODIUM_SRC "sodium/*.c" "sodium/*.h")
    target_sources(sodium-builtin PRIVATE ${SODIUM_SRC})
else()
    # Include assembly files for non-Windows (assuming x64/AVX)
    file(GLOB SODIUM_ASM "sodium/*.S")
    target_sources(sodium-builtin PRIVATE ${SODIUM_ASM})
endif()

# Suppress specific warnings for GNU Make
if(CMAKE_GENERATOR MATCHES "Unix Makefiles")
    target_compile_options(sodium-builtin PRIVATE
            -Wno-unused-parameter
            -Wno-unused-function
            -Wno-unknown-pragmas
            -Wno-unused-variable
            -Wno-type-limits
    )
endif()

# netcode library
add_library(netcode STATIC netcode/netcode.c netcode/netcode.h)
target_compile_definitions(netcode PRIVATE NETCODE_ENABLE_TESTS=1)

# reliable library
add_library(reliable STATIC reliable/reliable.c reliable/reliable.h)
target_compile_definitions(reliable PRIVATE RELIABLE_ENABLE_TESTS=1)

# tlsf library
add_library(tlsf STATIC tlsf/tlsf.c tlsf/tlsf.h)

# yojimbo library
file(GLOB YOJIMBO_HEADERS "include/*.h")
file(GLOB YOJIMBO_SOURCES "source/*.cpp")
add_library(yojimbo STATIC ${YOJIMBO_HEADERS} ${YOJIMBO_SOURCES})

# Define compile definitions based on configuration
target_compile_definitions(yojimbo PRIVATE
        $<$<CONFIG:Debug>:YOJIMBO_DEBUG NETCODE_DEBUG RELIABLE_DEBUG>
        $<$<CONFIG:Release>:YOJIMBO_RELEASE NETCODE_RELEASE RELIABLE_RELEASE>
)

# =====================================
# Define Executables
# =====================================

# Function to simplify executable creation
function(add_yojimbo_exe exe_name src_file)
    add_executable(${exe_name} ${src_file} shared.h)
    target_include_directories(${exe_name} PRIVATE ${COMMON_INCLUDE_DIRS})

    if(WIN32)
        target_link_libraries(${exe_name} PRIVATE yojimbo sodium-builtin tlsf netcode reliable)
    else()
        target_link_libraries(${exe_name} PRIVATE yojimbo sodium tlsf netcode reliable)
        target_link_directories(${exe_name} PRIVATE /opt/homebrew/lib)
    endif()

    # Set output directory for executables
    set_target_properties(${exe_name} PROPERTIES
            RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
    )
endfunction()

# Create executables
add_yojimbo_exe(client client.cpp)
add_yojimbo_exe(server server.cpp)
add_yojimbo_exe(loopback loopback.cpp)
add_yojimbo_exe(soak soak.cpp)

# Test executable
add_executable(test test.cpp)
target_compile_definitions(test PRIVATE SERIALIZE_ENABLE_TESTS=1)
target_include_directories(test PRIVATE ${COMMON_INCLUDE_DIRS})

if(WIN32)
    target_link_libraries(test PRIVATE yojimbo sodium-builtin tlsf netcode reliable)
else()
    target_link_libraries(test PRIVATE yojimbo sodium tlsf netcode reliable)
    target_link_directories(test PRIVATE /opt/homebrew/lib)
endif()

# Set output directory for test executable
set_target_properties(test PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
)

# =====================================
# Custom Clean Action
# =====================================

# Define a custom clean target to remove build files and outputs
if(WIN32)
    add_custom_target(clean_all
            COMMAND cmd /c "del /F /Q Makefile *.make *.txt *.zip *.tar.gz *.db *.opendb *.vcproj *.vcxproj *.vcxproj.user *.vcxproj.filters *.sln *.xcodeproj *.xcworkspace"
            COMMAND cmd /c "rmdir /S /Q obj ipch bin .vs Debug Release release cov-int docker\\yojimbo valgrind\\yojimbo docs xml"
            COMMENT "Cleaning all build files and output for Windows"
    )
else()
    add_custom_target(clean_all
            COMMAND find . -name .DS_Store -delete
            COMMAND rm -f Makefile *.make *.txt *.zip *.tar.gz *.db *.opendb *.vcproj *.vcxproj *.vcxproj.user *.vcxproj.filters *.sln *.xcodeproj *.xcworkspace
            COMMAND rm -rf obj ipch bin .vs Debug Release release cov-int docker/yojimbo valgrind/yojimbo docs xml
            COMMENT "Cleaning all build files and output for non-Windows"
    )
endif()
