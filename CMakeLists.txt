cmake_minimum_required(VERSION 3.16)
project(Project2 CXX C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ---------------------------------------------------------------
# Default to an optimized Release build (-O3 -DNDEBUG on GCC/MinGW).
# Override with -DCMAKE_BUILD_TYPE=Debug when you need to debug.
# ---------------------------------------------------------------
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# ---------------------------------------------------------------
# ccache (optional): speeds up clean rebuilds if it is installed
# ---------------------------------------------------------------
find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_PROGRAM})
    set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE_PROGRAM})
    message(STATUS "ccache       : ${CCACHE_PROGRAM}")
endif()

option(STATIC_RUNTIME "Link the MinGW runtime statically (exe needs no MinGW DLLs)" OFF)

set(PCPP ${CMAKE_SOURCE_DIR}/PcapPlusPlus)
set(NPCAP_SDK "E:/Code/Npcap-SDK" CACHE PATH "Npcap SDK folder")

# ---------------------------------------------------------------
# Helper: find a header anywhere under PcapPlusPlus/3rdParty and
# return the folder it lives in (works for any folder name/case).
# ---------------------------------------------------------------
function(find_header_dir out header)
    file(GLOB_RECURSE _found "${PCPP}/3rdParty/*/${header}")
    if(NOT _found)
        file(GLOB_RECURSE _found "${PCPP}/3rdParty/${header}")
    endif()
    list(GET _found 0 _first)
    if(_first)
        get_filename_component(_dir "${_first}" DIRECTORY)
        set(${out} "${_dir}" PARENT_SCOPE)
    else()
        set(${out} "" PARENT_SCOPE)
    endif()
endfunction()

find_header_dir(MD5_DIR md5.h)
find_header_dir(ENDIAN_DIR EndianPortable.h)
find_header_dir(LPNG_DIR light_pcapng.h)
find_header_dir(JSON_DIR json.hpp)

message(STATUS "Build type   : ${CMAKE_BUILD_TYPE}")
message(STATUS "MD5 dir      : ${MD5_DIR}")
message(STATUS "Endian dir   : ${ENDIAN_DIR}")
message(STATUS "LightPcapNg  : ${LPNG_DIR}")
message(STATUS "JSON dir     : ${JSON_DIR}")

# ---------------------------------------------------------------
# Include directories
# ---------------------------------------------------------------
include_directories(
    include
    ${PCPP}/Pcap++/header
    ${PCPP}/Pcap++/src
    ${PCPP}/Common++/header
    ${PCPP}/Common++/src
    ${PCPP}/Packet++/header
    ${PCPP}/3rdParty
    ${MD5_DIR}
    ${ENDIAN_DIR}
    ${LPNG_DIR}
    ${LPNG_DIR}/..
    ${JSON_DIR}
    ${JSON_DIR}/..
)

if(EXISTS "${NPCAP_SDK}/Include/pcap.h")
    include_directories("${NPCAP_SDK}/Include")
endif()

# ---------------------------------------------------------------
# Sources
# CONFIGURE_DEPENDS makes CMake notice added/removed files on the next build
# ---------------------------------------------------------------
# Common++ (everything except the OUI lookup, which needs a big dataset)
file(GLOB COMMON_SRCS CONFIGURE_DEPENDS "${PCPP}/Common++/src/*.cpp")
list(FILTER COMMON_SRCS EXCLUDE REGEX "OUILookup\\.cpp$")

# Packet++ (all protocol layers; TcpLayer and others depend on SSL, so keep them)
file(GLOB PACKET_SRCS CONFIGURE_DEPENDS "${PCPP}/Packet++/src/*.cpp")

# Pcap++ (file reading only, no live capture devices)
set(PCAP_SRCS
    ${PCPP}/Pcap++/src/PcapFileDevice.cpp
    ${PCPP}/Pcap++/src/PcapFilter.cpp
)
if(EXISTS "${PCPP}/Pcap++/src/PcapUtils.cpp")
    list(APPEND PCAP_SRCS ${PCPP}/Pcap++/src/PcapUtils.cpp)
endif()

# Third party: MD5 (needed by SSLHandshake) and LightPcapNg (C sources)
set(THIRD_SRCS "")
if(MD5_DIR)
    file(GLOB MD5_SRCS "${MD5_DIR}/md5.cpp")
    list(APPEND THIRD_SRCS ${MD5_SRCS})
endif()
if(LPNG_DIR)
    file(GLOB_RECURSE LPNG_SRCS "${PCPP}/3rdParty/*.c")
    list(FILTER LPNG_SRCS INCLUDE REGEX "[Ll]ight[Pp]cap[Nn]g/")
    list(FILTER LPNG_SRCS EXCLUDE REGEX "/([Tt]ests?|[Ee]xamples?|[Tt]ools?)/")
    list(APPEND THIRD_SRCS ${LPNG_SRCS})
endif()

# ---------------------------------------------------------------
# PcapPlusPlus as its own static library (compiled once, then cached)
# ---------------------------------------------------------------
add_library(pcpp STATIC ${COMMON_SRCS} ${PACKET_SRCS} ${PCAP_SRCS} ${THIRD_SRCS})
target_compile_options(pcpp PRIVATE
    $<$<COMPILE_LANGUAGE:CXX>:-Wno-deprecated-copy>
    -Wno-unused-parameter)

# ---------------------------------------------------------------
# Your program (only these 5 files recompile when you edit your code)
# ---------------------------------------------------------------
add_executable(Project2
    src/main.cpp
    src/pcapHandler.cpp
    src/data_hash_table.cpp
    src/aho_corasick.cpp
    src/cidr_blacklist.cpp
)
target_compile_options(Project2 PRIVATE
    -Wall -Wextra
    -Wno-deprecated-copy -Wno-unused-parameter)

target_link_libraries(Project2 PRIVATE pcpp ws2_32 iphlpapi)

# PcapFilter.cpp uses libpcap functions, so link Npcap's import libraries
if(EXISTS "${NPCAP_SDK}/Lib/x64/wpcap.lib")
    target_link_libraries(Project2 PRIVATE
        "${NPCAP_SDK}/Lib/x64/wpcap.lib"
        "${NPCAP_SDK}/Lib/x64/Packet.lib")
endif()

# ---------------------------------------------------------------
# Link-time optimization for your own code only (Release builds).
# Not applied to pcpp: it would slow the build a lot for little gain.
# ---------------------------------------------------------------
#
# OFF by default: with MinGW, LTO on Project2 alone fails to link against the
# non-LTO libpcpp.a (discarded-section errors on PcapFileReaderDevice thunks).
# If you enable it, it must be applied to pcpp as well.
option(ENABLE_LTO "Enable link-time optimization (Release only)" OFF)
if(ENABLE_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT IPO_OK OUTPUT IPO_MSG LANGUAGES CXX)
    if(IPO_OK)
        set_property(TARGET pcpp Project2 PROPERTY INTERPROCEDURAL_OPTIMIZATION_RELEASE ON)
    else()
        message(STATUS "LTO not supported: ${IPO_MSG}")
    endif()
endif()

# Optional: static MinGW runtime (configure with -DSTATIC_RUNTIME=ON)
if(STATIC_RUNTIME AND MINGW)
    target_link_options(Project2 PRIVATE -static-libgcc -static-libstdc++)
endif()