# Copyright 2021 Ingemar Hedvall
# SPDX-License-Identifier: MIT
cmake_minimum_required(VERSION 3.20)
project(utilLib
        VERSION 1.0
        DESCRIPTION "Common C++ library with utilities."
        LANGUAGES CXX)

option(BUILD_SHARED_LIBS "Static libraries are preferred" OFF)
option(BUILD_STATIC_LIBS "Static libraries are normally used" ON)
option(BUILD_DOC "If doxygen is installed, then build documentation in Release mode" ON)
if (NOT COMP_DIR)
    set(COMP_DIR "k:")
endif()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_DEBUG_POSTFIX d)

include(CMakePrintHelpers)
include(script/boost.cmake)
#include(script/zlib.cmake)
#include(script/openssl.cmake)
include(script/expat.cmake)
include(script/wxwidgets.cmake)
include(script/googletest.cmake)
include(script/doxygen.cmake)
include(script/flex.cmake)
include(script/bison.cmake)
#include(script/mqtt.cmake)
#include(script/mqtt_cpp.cmake)

if (BISON_FOUND)
    BISON_TARGET(SYSLOG_PARSER src/syslogparser.y ${CMAKE_CURRENT_SOURCE_DIR}/src/syslogparser.cpp)
    cmake_print_variables(BISON_SYSLOG_PARSER_DEFINED
            BISON_SYSLOG_PARSER_INPUT
            BISON_SYSLOG_PARSER_OUTPUT_SOURCE
            BISON_SYSLOG_PARSER_OUTPUT_HEADER
            BISON_SYSLOG_PARSER_OUTPUTS
            BISON_SYSLOG_PARSER_COMPILE_FLAGS)
endif()

if (FLEX_FOUND)
    FLEX_TARGET(SYSLOG_SCANNER src/syslogflexer.l ${CMAKE_CURRENT_SOURCE_DIR}/src/syslogflexer.cpp)
    cmake_print_variables(FLEX_SYSLOG_SCANNER_DEFINED
                          FLEX_SYSLOG_SCANNER_OUTPUTS
                          FLEX_SYSLOG_SCANNER_INPUT
                          FLEX_SYSLOG_SCANNER_OUTPUT_HEADER )
endif()

ADD_FLEX_BISON_DEPENDENCY(SYSLOG_SCANNER SYSLOG_PARSER)

add_library(util STATIC
        src/logfile.cpp src/logfile.h
        include/util/timestamp.h src/timestamp.cpp
        include/util/logconfig.h src/logconfig.cpp
        include/util/logging.h src/logging.cpp
        src/logconsole.cpp src/logconsole.h
        include/util/logmessage.h src/logmessage.cpp
        include/util/logstream.h src/logstream.cpp
        src/expatxml.h src/expatxml.cpp
        src/xmlnode.h src/xmlnode.cpp
        include/util/ixmlnode.h src/ixmlnode.cpp
        include/util/ixmlfile.h src/ixmlfile.cpp
        include/util/stringutil.h src/stringutil.cpp
        include/util/ilogger.h
        include/util/csvwriter.h src/csvwriter.cpp
        src/messagequeue.cpp src/messagequeue.h
        src/listenmessage.h src/listenmessage.cpp
        src/ilisten.cpp include/util/ilisten.h
        src/listenserver.cpp src/listenserver.h
        src/listenproxy.cpp src/listenproxy.h
        include/util/threadsafequeue.h
        src/listenserverconnection.h src/listenserverconnection.cpp
        src/listenconfig.cpp include/util/listenconfig.h
        src/listenclient.cpp src/listenclient.h
        src/listenlogger.cpp src/listenlogger.h
        src/writexml.cpp src/writexml.h
        src/stringparser.cpp include/util/stringparser.h
        src/gnuplot.cpp include/util/gnuplot.h
        src/tempdir.cpp include/util/tempdir.h
        src/syslogmessage.cpp include/util/syslogmessage.h
        src/syslog.cpp src/syslog.h src/ilogger.cpp
        ${BISON_SYSLOG_PARSER_OUTPUTS}
        ${FLEX_SYSLOG_SCANNER_OUTPUTS}
        src/syslogscanner.cpp src/syslogscanner.h
        src/structureddata.cpp include/util/structureddata.h
        src/isyslogserver.cpp include/util/isyslogserver.h
        src/udpsyslogserver.cpp src/udpsyslogserver.h
        src/utilfactory.cpp include/util/utilfactory.h
        src/listenconsole.cpp src/listenconsole.h)

#get_target_property(PAHO_C_INCLUDE_DIRS eclipse-paho-mqtt-c::paho-mqtt3as-static INTERFACE_INCLUDE_DIRECTORIES)

target_include_directories(util PUBLIC
        $<INSTALL_INTERFACE:include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        )

target_include_directories(util PRIVATE ${Boost_INCLUDE_DIRS})
target_include_directories(util PRIVATE ${EXPAT_INCLUDE_DIRS})
#target_include_directories(util PRIVATE ${ZLIB_INCLUDE_DIRS})
#target_include_directories(util PRIVATE ${OPENSSL_INCLUDE_DIR})
target_include_directories(util PRIVATE ${FLEX_INCLUDE_DIRS})
#target_include_directories(util PRIVATE ${PAHO_C_INCLUDE_DIRS})

cmake_print_properties(TARGETS util PROPERTIES INCLUDE_DIRECTORIES)

target_compile_definitions(util PRIVATE XML_STATIC)

if (MSVC)
    target_compile_definitions(util PRIVATE _WIN32_WINNT=0x0A00)
endif()

set(UTIL_PUBLIC_HEADERS
    include/util/csvwriter.h
    include/util/ilogger.h
    include/util/ixmlnode.h
    include/util/logconfig.h
    include/util/logging.h
    include/util/logmessage.h
    include/util/stringutil.h
    include/util/timestamp.h
)

set_target_properties(util PROPERTIES PUBLIC_HEADER "${UTIL_PUBLIC_HEADERS}")
get_target_property(PH util PUBLIC_HEADER)
message( STATUS "UTIL Target Includes: " "${PH}")

add_subdirectory(listend)
if (WIN32)
    add_subdirectory(serviced)
endif()

if (wxWidgets_FOUND )
    add_subdirectory(listenviewer)
endif()
if (wxWidgets_FOUND AND WIN32)
    add_subdirectory(serviceexplorer)
endif()

if (GTEST_FOUND)
    enable_testing()
    add_subdirectory(test)
endif()

if (DOXYGEN_FOUND AND (CMAKE_BUILD_TYPE MATCHES "^[Rr]elease") )
    set(DOXYGEN_RECURSIVE NO)
    set(DOXYGEN_REPEAT_BRIEF NO)
    set(DOXYGEN_PROJECT_NAME "UTILLib")
    set(DOXYGEN_HTML_EXTRA_STYLESHEET doxygen/utillib.css)
    set(DOXYGEN_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/docs/manual)
    doxygen_add_docs( doc_libutil ALL
            include/util doxygen
            COMMENT "Util Library generating API documentation with Doxygen")
endif()

install(TARGETS util
        EXPORT UtilLibTargets
        LIBRARY DESTINATION util/lib
        ARCHIVE DESTINATION util/lib
        RUNTIME DESTINATION util/bin
        PUBLIC_HEADER DESTINATION util/include/util )

#include(CMakePackageConfigHelpers)
#write_basic_package_version_file(
#        UtilLibConfigVersion.cmake
#        VERSION ${PACKAGE_VERSION}
#        COMPATIBILITY AnyNewerVersion)
#
#install(EXPORT UtilLibTargets
#        FILE UtilLibTargets.cmake
#        NAMESPACE UtilLib::
#        DESTINATION lib/cmake/UtilLib
#        )

#export(TARGETS util NAMESPACE UtilLib:: FILE UtilLibTargets.cmake)