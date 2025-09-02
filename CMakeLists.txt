cmake_minimum_required(VERSION 3.28)
project(uSockets)

set(CMAKE_CXX_STANDARD 17)

if(WIN32)
    set(WINDOWS TRUE)
else()
    set(WINDOWS FALSE)
endif()

# 选项配置
option(WITH_LTO "Enable Link Time Optimization" OFF)
option(WITH_BORINGSSL "Enable BoringSSL support" OFF)
option(WITH_OPENSSL "Enable OpenSSL support" ON)
option(WITH_WOLFSSL "Enable WolfSSL support" OFF)
option(WITH_IO_URING "Build with io_uring" OFF)
option(WITH_LIBUV "Build with libuv" OFF)
option(WITH_ASIO "Build with Boost Asio" OFF)
option(WITH_GCD "Build with libdispatch" OFF)
option(WITH_ASAN "Build with address sanitizer" OFF)
option(WITH_QUIC "Build with QUIC support" OFF)

add_definitions(-DCMAKE_BUILD_USOCKETS)

set(WITH_OPENSSL ON)

message("WITH_OPENSSL  ${WITH_OPENSSL}")


if(DEFINED CMAKE_CURRENT_USER_DIR)  # 额外检查变量是否定义（避免极端环境问题）
    set(USER_HOME_DIR "${CMAKE_CURRENT_USER_DIR}")
else()
    # 降级处理：若变量未定义（理论上 3.14+ 不会出现），通过环境变量兼容
    if(WIN32)
        set(USER_HOME_DIR "$ENV{USERPROFILE}")
    else()
        set(USER_HOME_DIR "$ENV{HOME}")
    endif()
endif()

if(EXISTS "${USER_HOME_DIR}")
    message(STATUS "current user home dir: ${USER_HOME_DIR}")
else()
    message(WARNING "can't found current user home dir: ${USER_HOME_DIR}")
endif()


if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CDEPS_DIR "${USER_HOME_DIR}/cdeps/debug")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address -g  -O0 ")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -g -O0 ")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
else ()
    set(CDEPS_DIR "${USER_HOME_DIR}/cdeps/release")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3 ")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 ")
    if(WITH_LTO AND NOT WINDOWS)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -flto")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -flto")
    endif()
endif()




set(DEPS_LIBS "")
set(USOCKETS_SOURCES
    src/libusockets.h
    src/context.c
    src/bsd.c
    src/loop.c
    src/quic.c
    src/quic.h
    src/socket.c
    src/udp.c
    src/crypto/openssl.c
    src/crypto/sni_tree.cpp
    src/eventing/asio.cpp
    src/eventing/epoll_kqueue.c
    src/eventing/gcd.c
    src/eventing/libuv.c
    src/io_uring/io_context.c
    src/io_uring/io_loop.c
    src/io_uring/io_socket.c
)



## use this tool build and install openssl to cdeps
## https://github.com/gubaojian/opensslcompile/tree/main/pc/openssl/3.5.1

if(WITH_OPENSSL)

    add_definitions(-DLIBUS_USE_OPENSSL)
    set(OPENSSL_USE_STATIC_LIBS TRUE)
    find_package(OpenSSL 3.5.1 REQUIRED CONFIG
            PATHS "${CDEPS_DIR}/openssl/3.5.1/lib/cmake/OpenSSL"
            NO_DEFAULT_PATH)
    list(APPEND DEPS_LIBS OpenSSL::SSL OpenSSL::Crypto)
    include_directories(${OPENSSL_INCLUDE_DIR})
else()
    add_definitions(-DLIBUS_NO_SSL)
endif()

if(WITH_IO_URING)
    add_definitions(-DLIBUS_USE_IO_URING)
    list(APPEND DEPS_LIBS "/usr/lib/liburing.a")
endif()

if(WITH_LIBUV)
    add_definitions(-DLIBUS_USE_LIBUV)
    find_package(LibUV REQUIRED)
    set(EVENT_LIBS ${LIBUV_LIBRARIES})
    include_directories(${LIBUV_INCLUDE_DIRS})
endif()

if(WITH_ASIO)
    add_definitions(-DLIBUS_USE_ASIO)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread -std=c++14")
    set(EVENT_LIBS stdc++ pthread)
    file(GLOB ASIO_SOURCES "src/eventing/asio.cpp")
endif()

if(WITH_GCD)
    add_definitions(-DLIBUS_USE_GCD)
    set(EVENT_LIBS "-framework CoreFoundation")
endif()

if(WITH_QUIC)
    add_definitions(-DLIBUS_USE_QUIC)
    include_directories("src" "lsquic/include")
    set(QUIC_LIBS
            "uSockets.a"
            "lsquic/src/liblsquic/liblsquic.a"
            "z"
            "m"
            "pthread"
    )
else()
    include_directories("src")
    set(QUIC_LIBS "uSockets.a")
endif()


message(STATUS "CDEPS_DIR dir: ${CDEPS_DIR}")
message(STATUS "CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS}")
message(STATUS "DEPS_LIBS dir: ${DEPS_LIBS}")


set(USOCKETS_INCLUDE_DIRS  "${CMAKE_SOURCE_DIR}/src")

include_directories(${USOCKETS_INCLUDE_DIRS})

add_library(uSockets STATIC ${USOCKETS_SOURCES})
target_link_libraries(uSockets PRIVATE
        ${DEPS_LIBS}
)

add_executable(
    http_server
    examples/http_server.c
)

target_link_libraries(
        http_server
        PRIVATE
        uSockets
        ${DEPS_LIBS}
)