# Author: enzo418
# Description: Simple rewrite of the Makefile present here. Exports the
#              uSockets without SSL library as "usocketslib".
# ----
cmake_minimum_required(VERSION 3.12)

project(uSockets VERSION 1.0.0)

# options
# -------
option(WITH_OPENSSL "Enable SSL support - Not compatible with WOLFSSL option")
option(WITH_WOLFSSL "Enable SSL support - Not compatible with OPENSSL option")

option(WITH_QUIC "Use QUIC protocol")
option(WITH_ASIO "Use ASIO")

option(usocket_INSTALL "Generate the install target" OFF)

# src files
# ---------
file(GLOB SRCS 
        src/*.c 
        # there is no problem in including these files below as they
        # hide the code behind c macros with their respective flag.
        src/eventing/*.c  
        src/crypto/*.c 
        src/crypto/*.cpp
)

# includes
# ---------
include_directories(src)

# library
# -------
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3")

add_library(usocketslib ${SRCS})

# LTO optimization
# -----------------
include(CheckIPOSupported)
check_ipo_supported(RESULT supported OUTPUT error)

if(supported)
    message(STATUS "LTO enabled")
    set_property(TARGET usocketslib PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
else()
    message(STATUS "LTO not supported: <${error}>")
endif()

# clangd
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# WITH_BORINGSSL => unsopported (we need to fork it and add cmake support)

# Applied options
# ---------------
if(WITH_OPENSSL)
        # RECOMMENDED! WITH_OPENSSL=ON enables OpenSSL 1.1+ support
        find_package(OpenSSL REQUIRED)
        
        include_directories(${OPENSSL_INCLUDE_DIR})

        target_compile_definitions(usocketslib PUBLIC LIBUS_USE_OPENSSL)

        # SSL -> system openssl installation (e.g. libssl-dev)
        # crypto -> uSockets -> crypto library
        target_link_libraries(usocketslib ${OPENSSL_LIBRARIES} "-lcrypto" "-lstdc++")
        message(STATUS "With OPENSSL")
elseif(WITH_WOLFSSL)
        # Not recommended, uSockets seems to miss something while using this
        # WITH_WOLFSSL=ON enables WolfSSL 4.2.0 support (mutually exclusive with OpenSSL)
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(wolfssl REQUIRED IMPORTED_TARGET wolfssl)
        
        include_directories(${WOLFSSL_INCLUDE_DIRS})
        target_compile_definitions(usocketslib PUBLIC LIBUS_USE_WOLFSSL)
        target_link_libraries(usocketslib PkgConfig::wolfssl)
        message(STATUS "With WOLFSSL")
else() # NO SSL
        target_compile_definitions(usocketslib PUBLIC LIBUS_NO_SSL)
        message(STATUS "SSL Disabled")
endif()

if(WITH_BORINGSSL OR WITH_ASIO OR WITH_QUIC)
        # add pthread
        set(THREADS_PREFER_PTHREAD_FLAG ON)
        find_package(Threads REQUIRED)
        target_link_libraries(usocketslib PRIVATE Threads::Threads)
endif()

if(WITH_ASIO)
        target_sources(usocketslib PRIVATE src/eventing/asio.cpp)
        target_compile_definitions(usocketslib PUBLIC LIBUS_USE_ASIO)
endif()

if(WITH_QUIC)
        set(LSQUIC_SHARED_LIB OFF) # build static lib (without boring ssl)
                                   # set it ON if Boring is installed, run find_packge before!
        target_compile_definitions(usocketslib PUBLIC LIBUS_USE_QUIC)
        include_directories(lsquic/include)
        add_subdirectory(lsquic)
        target_link_libraries(usocketslib "-lz" "-lm" lsquic)
endif()

set (CMAKE_CXX_STANDARD 17) # clangd cries if we remove this since the code uses std::string_view

# ------------------------------------------------------ #
#                 install lib and headers                #
# ------------------------------------------------------ #
if (usocket_INSTALL)
    install(TARGETS usocketslib
            EXPORT uSocketsTargets
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_LIBDIR}
    )
endif()