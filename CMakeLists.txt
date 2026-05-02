cmake_minimum_required(VERSION 3.30)
project(mumlib)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -O3")
add_definitions(-DOPT_TLS_OPENSSL)
if (MSYS)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -static-libstdc++ -static-libgcc -static")
  add_definitions(-DLOG4CPP_FIX_ERROR_COLLISION=1 -D__USE_W32_SOCKETS)
elseif (NOT APPLE)
  add_definitions(-D_POSIX_C_SOURCE=200112L)
endif()


if (APPLE)
    link_directories(/opt/homebrew/lib) # Include the default homebrew library directory
endif()

find_package(PkgConfig REQUIRED)

if (NOT MSYS)
find_package(Boost REQUIRED)
endif()

find_package(OpenSSL REQUIRED)
find_package(Protobuf CONFIG REQUIRED)

pkg_check_modules(LOG4CPP "log4cpp" REQUIRED IMPORTED_TARGET)
pkg_check_modules(Speex "speex" REQUIRED IMPORTED_TARGET)
pkg_check_modules(SpeexDSP "speexdsp" REQUIRED IMPORTED_TARGET)
pkg_check_modules(Opus "opus" REQUIRED IMPORTED_TARGET)

if (MSYS)
include_directories(~/boost_1_72_0)
elseif (NOT APPLE)
INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIR})
endif()

include_directories(${OPENSSL_INCLUDE_DIRS})
#include_directories(${PROTOBUF_INCLUDE_DIRS})
include_directories(${OPUS_INCLUDE_DIRS})
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${LOG4CPP_INCLUDE_DIRS})
include_directories(include)

set(MUMLIB_PUBLIC_HEADERS include/mumlib.hpp include/mumlib/VarInt.hpp)

set(MUMLIB_PRIVATE_HEADERS
        include/mumlib/Callback.hpp
        include/mumlib/CryptState.hpp
        include/mumlib/Transport.hpp
        include/mumlib/Audio.hpp
        include/mumlib/enums.hpp
        )

set(MUMLIB_SRC
        src/mumlib.cpp
        src/Callback.cpp
        src/CryptState.cpp
        src/VarInt.cpp
        src/Transport.cpp
        src/Audio.cpp
        )

set(SPEEX_LIBRARIES
        speexdsp
        )

add_library(mumlib STATIC ${MUMLIB_SRC} ${MUMLIB_PUBLIC_HEADERS} ${MUMLIB_PRIVATE_HEADERS})
target_include_directories(mumlib PUBLIC include)

protobuf_generate(TARGET mumlib PROTOS Mumble.proto)

target_link_libraries(mumlib INTERFACE ${Boost_LIBRARIES})
target_link_libraries(mumlib PRIVATE
        PkgConfig::Speex PkgConfig::SpeexDSP PkgConfig::Opus
        OpenSSL::SSL OpenSSL::Crypto
        PkgConfig::LOG4CPP)
target_link_libraries(mumlib PRIVATE protobuf::libprotobuf)
if (MSYS)
target_link_libraries(mumlib PRIVATE ws2_32 winmm z)
endif ()

set_target_properties(mumlib PROPERTIES PUBLIC_HEADER "${MUMLIB_PUBLIC_HEADERS}")

add_executable(mumlib_example mumlib_example.cpp)

if (MSYS)
target_link_libraries(mumlib_example mumlib ws2_32)
else ()
target_link_libraries(mumlib_example mumlib)
endif ()

install(TARGETS DESTINATION mumlib LIBRARY DESTINATION lib PUBLIC_HEADER DESTINATION include/mumlib)
