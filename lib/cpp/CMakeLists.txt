#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements. See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership. The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License. You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied. See the License for the
# specific language governing permissions and limitations
# under the License.
#

include_directories(src)

# SYSLIBS contains libraries that need to be linked to all lib targets
set(SYSLIBS "")
set(package_deps Boost)
set(boost_components)

list(APPEND SYSLIBS Boost::boost)

# Create the thrift C++ library
set( thriftcpp_SOURCES
   src/thrift/TApplicationException.cpp
   src/thrift/TOutput.cpp
   src/thrift/async/TAsyncChannel.cpp
   src/thrift/async/TConcurrentClientSyncInfo.h
   src/thrift/async/TConcurrentClientSyncInfo.cpp
   src/thrift/concurrency/ThreadManager.cpp
   src/thrift/concurrency/TimerManager.cpp
   src/thrift/concurrency/Util.cpp
   src/thrift/processor/PeekProcessor.cpp
   src/thrift/protocol/TBase64Utils.cpp
   src/thrift/protocol/TDebugProtocol.cpp
   src/thrift/protocol/TJSONProtocol.cpp
   src/thrift/protocol/TMultiplexedProtocol.cpp
   src/thrift/protocol/TProtocol.cpp
   src/thrift/transport/TTransportException.cpp
   src/thrift/transport/TFDTransport.cpp
   src/thrift/transport/TSimpleFileTransport.cpp
   src/thrift/transport/THttpTransport.cpp
   src/thrift/transport/THttpClient.cpp
   src/thrift/transport/THttpServer.cpp
   src/thrift/transport/TSocket.cpp
   src/thrift/transport/TSocketPool.cpp
   src/thrift/transport/TServerSocket.cpp
   src/thrift/transport/TTransportUtils.cpp
   src/thrift/transport/TBufferTransports.cpp
   src/thrift/server/TConnectedClient.cpp
   src/thrift/server/TServerFramework.cpp
   src/thrift/server/TSimpleServer.cpp
   src/thrift/server/TThreadPoolServer.cpp
   src/thrift/server/TThreadedServer.cpp
)

# These files don't work on Windows CE as there is no pipe support
# TODO: These files won't work with UNICODE support on windows. If fixed this can be re-added.
if (NOT WINCE)
    list(APPEND thriftcpp_SOURCES
       src/thrift/transport/TPipe.cpp
       src/thrift/transport/TPipeServer.cpp
       src/thrift/transport/TFileTransport.cpp
    )
endif()


if (WIN32)
    list(APPEND thriftcpp_SOURCES
        src/thrift/windows/TWinsockSingleton.cpp
        src/thrift/windows/SocketPair.cpp
        src/thrift/windows/GetTimeOfDay.cpp
        src/thrift/windows/WinFcntl.cpp
    )
    if(NOT WINCE)
        # This file uses pipes so it currently won't work on Windows CE
        list(APPEND thriftcpp_SOURCES
            src/thrift/windows/OverlappedSubmissionThread.cpp
        )
    endif()
else()
    # These files evaluate to nothing on Windows, so omit them from the
    # Windows build
    list(APPEND thriftcpp_SOURCES
        src/thrift/VirtualProfiling.cpp
        src/thrift/server/TServer.cpp
    )
endif()

# If OpenSSL is not found just ignore the OpenSSL stuff
if(WITH_OPENSSL)
    hunter_add_package(OpenSSL)
    find_package(OpenSSL REQUIRED)
    list( APPEND thriftcpp_SOURCES
       src/thrift/transport/TSSLSocket.cpp
       src/thrift/transport/TSSLServerSocket.cpp
    )
    include_directories(SYSTEM "${OPENSSL_INCLUDE_DIR}")
    list(APPEND SYSLIBS OpenSSL::SSL OpenSSL::Crypto)
    list(APPEND package_deps OpenSSL)
endif()

# WITH_*THREADS selects which threading library to use
if(WITH_BOOSTTHREADS)
    set( thriftcpp_threads_SOURCES
        src/thrift/concurrency/BoostThreadFactory.cpp
        src/thrift/concurrency/BoostMonitor.cpp
        src/thrift/concurrency/BoostMutex.cpp
    )
    list(APPEND boost_components thread)
elseif(UNIX AND NOT WITH_STDTHREADS)
    if(ANDROID)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
    else()
        list(APPEND SYSLIBS pthread)
    endif()
    set( thriftcpp_threads_SOURCES
        src/thrift/concurrency/PosixThreadFactory.cpp
        src/thrift/concurrency/Mutex.cpp
        src/thrift/concurrency/Monitor.cpp
    )
else()
    if(UNIX)
        if(ANDROID)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
        else()
            list(APPEND SYSLIBS pthread)
        endif()
    endif()
    set( thriftcpp_threads_SOURCES
        src/thrift/concurrency/StdThreadFactory.cpp
        src/thrift/concurrency/StdMutex.cpp
        src/thrift/concurrency/StdMonitor.cpp
    )
endif()

# Thrift non blocking server
set( thriftcppnb_SOURCES
    src/thrift/server/TNonblockingServer.cpp
    src/thrift/transport/TNonblockingServerSocket.cpp
    src/thrift/transport/TNonblockingSSLServerSocket.cpp
    src/thrift/async/TAsyncProtocolProcessor.cpp
    src/thrift/async/TEvhttpServer.cpp
    src/thrift/async/TEvhttpClientChannel.cpp
)

# Thrift zlib server
set( thriftcppz_SOURCES
    src/thrift/transport/TZlibTransport.cpp
    src/thrift/protocol/THeaderProtocol.cpp
    src/thrift/transport/THeaderTransport.cpp
    src/thrift/protocol/THeaderProtocol.cpp
    src/thrift/transport/THeaderTransport.cpp
)

# Thrift Qt4 server
set( thriftcppqt_SOURCES
    src/thrift/qt/TQIODeviceTransport.cpp
    src/thrift/qt/TQTcpServer.cpp
)

# Contains the thrift specific ADD_LIBRARY_THRIFT and TARGET_LINK_LIBRARIES_THRIFT
include(ThriftMacros)

ADD_LIBRARY_THRIFT(thrift ${thriftcpp_SOURCES} ${thriftcpp_threads_SOURCES})
if(WIN32)
    TARGET_LINK_LIBRARIES_THRIFT(thrift PUBLIC ${SYSLIBS} ws2_32)
else()
  TARGET_LINK_LIBRARIES_THRIFT(thrift PUBLIC ${SYSLIBS})
endif()

if(WITH_LIBEVENT)
    hunter_add_package(Libevent)
    find_package(Libevent CONFIG REQUIRED)  # Libevent comes with CMake support form upstream
    include_directories(SYSTEM ${LIBEVENT_INCLUDE_DIRS})

    ADD_LIBRARY_THRIFT(thriftnb ${thriftcppnb_SOURCES})
    TARGET_LINK_LIBRARIES_THRIFT(thriftnb ${SYSLIBS} Libevent::event_core Libevent::event_extra)
    TARGET_LINK_LIBRARIES_THRIFT_AGAINST_THRIFT_LIBRARY(thriftnb thrift)
    list(APPEND package_deps Libevent)
endif()

if(WITH_ZLIB)
    hunter_add_package(ZLIB)
    find_package(ZLIB CONFIG REQUIRED)
    include_directories(SYSTEM ${ZLIB_INCLUDE_DIRS})

    ADD_LIBRARY_THRIFT(thriftz ${thriftcppz_SOURCES})
    TARGET_LINK_LIBRARIES_THRIFT(thriftz ${SYSLIBS} ZLIB::zlib)
    TARGET_LINK_LIBRARIES_THRIFT_AGAINST_THRIFT_LIBRARY(thriftz thrift)
    list(APPEND package_deps ZLIB)
endif()

if(WITH_QT4)
    set(CMAKE_AUTOMOC ON)
    find_package(Qt4 REQUIRED COMPONENTS QtCore QtNetwork)
    ADD_LIBRARY_THRIFT(thriftqt ${thriftcppqt_SOURCES})
    TARGET_LINK_LIBRARIES_THRIFT(thriftqt ${SYSLIBS} Qt4::QtCore Qt4::QtNetwork)
    TARGET_LINK_LIBRARIES_THRIFT_AGAINST_THRIFT_LIBRARY(thriftqt thrift)
endif()

if(WITH_QT5)
    # Qt5 has its own directory to avoid conflict with Qt4 caused by CMAKE_AUTOMOC
    add_subdirectory(src/thrift/qt)
endif()

if(MSVC)
    add_definitions("-DUNICODE -D_UNICODE")
endif()

add_definitions("-D__STDC_LIMIT_MACROS")

# Install the headers
install(DIRECTORY "src/thrift" DESTINATION "${INCLUDE_INSTALL_DIR}"
    FILES_MATCHING PATTERN "*.h" PATTERN "*.tcc")
# Copy config.h file
install(DIRECTORY "${CMAKE_BINARY_DIR}/thrift" DESTINATION "${INCLUDE_INSTALL_DIR}"
    FILES_MATCHING PATTERN "*.h")

# Installation (https://github.com/forexample/package-example)

# Introduce variables:
# * CMAKE_INSTALL_LIBDIR
# * CMAKE_INSTALL_BINDIR
# * CMAKE_INSTALL_INCLUDEDIR
include(GNUInstallDirs)

# Layout. This works for all platforms:
#   * <prefix>/lib*/cmake/<PROJECT-NAME>
#   * <prefix>/lib*/
#   * <prefix>/include/
set(config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# Include module with fuction 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Configure '<PROJECT-NAME>ConfigVersion.cmake'
# Use:
#   * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Configure '<PROJECT-NAME>Config.cmake'
# Use variables:
#   * TARGETS_EXPORT_NAME
#   * PROJECT_NAME
configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/build/cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

# Targets:
#   * <prefix>/lib/libthrift.a
#   * <prefix>/lib/libthrift.so
#   * header location after install: <prefix>/include/thrift/Thrift.h
#   * headers can be included by C++ code `#include <thrift/Thrift.h>`
install(
    TARGETS ${thrift_lib_targets}
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

# Headers:
#   * src/thrift/Thrift.h -> <prefix>/include/thrift/Thrift.h
install(
    DIRECTORY "src/thrift"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    FILES_MATCHING PATTERN "*.h"
)

# Config
#   * <prefix>/lib/cmake/thrift/thriftConfig.cmake
#   * <prefix>/lib/cmake/thrift/thriftConfigVersion.cmake
install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

# Config
#   * <prefix>/lib/cmake/thrift/thriftTargets.cmake
install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)

if(BUILD_TESTING)
    add_subdirectory(test)
endif()
