cmake_minimum_required ( VERSION 3.16 )

# Parse version number from the header file
file(READ ${CMAKE_CURRENT_SOURCE_DIR}/liveMedia/include/liveMedia_version.hh contents)
string(REGEX MATCH "define LIVEMEDIA_LIBRARY_VERSION_STRING	\"([0-9\.]+)\"" _ ${contents})
set(project_version ${CMAKE_MATCH_1})
if(NOT DEFINED PROJECT_NAME)
    set(live555_master_project TRUE)
endif()

project(live555
    LANGUAGES C CXX
    VERSION ${project_version}
)
message(STATUS "live555 version: ${project_version}")

# Decide on the standard to use
if(MDARRAY_CXX_STANDARD STREQUAL "17")
  if("cxx_std_17" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    message(STATUS "Using C++17 standard")
    set(NO_STD_LIB TRUE)
    set(CMAKE_CXX_STANDARD 17)
  else()
    message(FATAL_ERROR "Requested MDARRAY_CXX_STANDARD \"17\" not supported by provided C++ compiler")
  endif()
elseif(MDARRAY_CXX_STANDARD STREQUAL "14")
  if("cxx_std_14" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    message(STATUS "Using C++14 standard")
    set(NO_STD_LIB TRUE)
    set(CMAKE_CXX_STANDARD 14)
  else()
    message(FATAL_ERROR "Requested MDARRAY_CXX_STANDARD \"14\" not supported by provided C++ compiler")
  endif()
elseif(MDARRAY_CXX_STANDARD STREQUAL "11")
  message(STATUS "Using C++11 standard")
  set(NO_STD_LIB TRUE)
  set(CMAKE_CXX_STANDARD 11)
elseif(MDARRAY_CXX_STANDARD STREQUAL "20")
  if("cxx_std_20" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    message(STATUS "Using C++20 standard")
    set(CMAKE_CXX_STANDARD 20)
  else()
    message(FATAL_ERROR "Requested MDARRAY_CXX_STANDARD \"20\" not supported by provided C++ compiler")
  endif()
else()
  if("cxx_std_20" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    set(CMAKE_CXX_STANDARD 20)
    message(STATUS "Detected support for C++20 standard")
  elseif("cxx_std_17" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    set(NO_STD_LIB TRUE)
    set(CMAKE_CXX_STANDARD 17)
    message(STATUS "Detected support for C++17 standard")
  elseif("cxx_std_14" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    set(NO_STD_LIB TRUE)
    set(CMAKE_CXX_STANDARD 14)
    message(STATUS "Detected support for C++14 standard")
  elseif("cxx_std_11" IN_LIST CMAKE_CXX_COMPILE_FEATURES)
    set(NO_STD_LIB TRUE)
    set(CMAKE_CXX_STANDARD 11)
    message(STATUS "Newest supported C++ standard detected is C++11.")
  else()
    message(FATAL_ERROR "Cannot detect CXX_STANDARD of C++11 or newer.")
  endif()
endif()

list(APPEND CMAKE_MODULE_PATH ${live555_SOURCE_DIR}/cmake)
include(TargetVersion)

option(LIVE555_EPOLL_SUPPORT "Enable epoll task scheduler (Windows also)" ON)
option(LIVE555_BUILD_EXAMPLES "Build examples and test programs" ${live555_master_project})
option(LIVE555_ENABLE_OPENSSL "Build with OpenSSL support (rtsps, SRTP)." ON)
option(LIVE555_MONOLITH_BUILD "Build all modules as one monolithic library" OFF)
option(LIVE555_BUILD_SHARED_LIBS "Build shared library" OFF)
option(LIVE555_BUILD_EMSCRIPTEN "Build emscripten" OFF)
option(LIVE555_INSTALL "Install live555" OFF)

macro(get_WIN32_WINNT version)
    if (WIN32 AND CMAKE_SYSTEM_VERSION)
        set(ver ${CMAKE_SYSTEM_VERSION})
        string(REGEX REPLACE "^([0-9])[.]([0-9]).*" "0\\10\\2" ver ${ver})
        set(${version} "0x${ver}")
    endif()
endmacro()

if (WIN32)
get_WIN32_WINNT(ver)
add_definitions(-DWIN32_WINNT=${ver})
endif ()

if ( CMAKE_COMPILER_IS_GNUCXX )
   set ( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fpermissive" )
endif ()

if ( LIVE555_BUILD_SHARED_LIBS )
   set ( BUILD_SHARED_LIBS TRUE )
else ()
   set ( BUILD_SHARED_LIBS FALSE )
endif ()

add_library(live555_cxx_flags INTERFACE)
if(WIN32)
    target_compile_definitions(live555_cxx_flags INTERFACE
        LOCALE_NOT_USED
        SOCKLEN_T=int
    )
    if(BUILD_SHARED_LIBS)
        set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
        set(CMAKE_DEBUG_POSTFIX d)
    endif()
else()
    target_compile_definitions(live555_cxx_flags INTERFACE
        BSD=1
        SOCKLEN_T=socklen_t
    )
endif()

#if ( LIVE555_ENABLE_OPENSSL )
#   message(STATUS "find package OpenSSL.")
#   set(EMSCRIPTEN_ROOT_PATH "$ENV{SYSROOT}")
#   if(WIN32)
#     find_package(OpenSSL REQUIRED COMPONENTS Crypto SSL)
#     target_link_libraries(live555_cxx_flags INTERFACE
#        OpenSSL::SSL
#        OpenSSL::Crypto
#     )
#   else ()
#     find_package(OpenSSL 1.1.1 REQUIRED COMPONENTS Crypto SSL)
#     target_link_libraries(live555_cxx_flags INTERFACE
#       OpenSSL::Crypto
#       OpenSSL::SSL
#     )
#   endif()
#else()
    message(STATUS "build without OpenSSL.")
    target_compile_definitions(live555_cxx_flags INTERFACE NO_OPENSSL=1)
#endif()

# if ( NOT DEFINED CMAKE_GETIFADDRS )
#     add_definitions(-DNO_GETIFADDRS)
# endif()

# if ( NOT DEFINED CMAKE_OPENSSL )
#     add_definitions(-DNO_OPENSSL)
# endif()

if(LIVE555_INSTALL)
   install(TARGETS live555_cxx_flags EXPORT live555Target)
endif()

if ( LIVE555_BUILD_EMSCRIPTEN )
   set ( BUILD_EMSCRIPTEN TRUE )
endif ()

if ( BUILD_EMSCRIPTEN )
# If not found by above search, locate using the EMSCRIPTEN environment variable.
if ("${EMSCRIPTEN_ROOT_PATH}" STREQUAL "")
   set(EMSCRIPTEN_ROOT_PATH "$ENV{EMSCRIPTEN}")
endif()

# Abort if not found.
if ("${EMSCRIPTEN_ROOT_PATH}" STREQUAL "")
   message(FATAL_ERROR "Could not locate the Emscripten compiler toolchain directory! Either set the EMSCRIPTEN environment variable, or pass -DEMSCRIPTEN_ROOT_PATH=xxx to CMake to explicitly specify the location of the compiler!")
endif()

if(WIN32)
    set(EMCC_SUFFIX ".bat")
else()
    set(EMCC_SUFFIX "")
endif()

if ( USE_DEPRECATED_GETHOSTBYNAME )
   set ( USE_DEPRECATED_GETHOSTBYNAME TRUE )
endif ( USE_DEPRECATED_GETHOSTBYNAME )

# Check version
set(EMSCRIPTEN_REQUIRED_VERSION 1.21.4)
if(EXISTS ${EMSCRIPTEN_ROOT_PATH}/emscripten-version.txt)
   file(READ ${EMSCRIPTEN_ROOT_PATH}/emscripten-version.txt EMSCRIPTEN_VERSION)
   string(STRIP ${EMSCRIPTEN_VERSION} EMSCRIPTEN_VERSION)
   string(REPLACE "\"" "" EMSCRIPTEN_VERSION ${EMSCRIPTEN_VERSION})
   message(STATUS "Found Emscripten [${EMSCRIPTEN_VERSION}]")
   if(EMSCRIPTEN_VERSION VERSION_LESS ${EMSCRIPTEN_REQUIRED_VERSION})
      message(FATAL_ERROR "Emscripten >= ${EMSCRIPTEN_REQUIRED_VERSION} is required.")
   endif()
   set (CMAKE_C_COMPILER ${EMSCRIPTEN_ROOT_PATH}/emcc${EMCC_SUFFIX})
   set (CMAKE_CXX_COMPILER ${EMSCRIPTEN_ROOT_PATH}/em++${EMCC_SUFFIX})
   set (CMAKE_RANLIB ${EMSCRIPTEN_ROOT_PATH}/emranlib${EMCC_SUFFIX} CACHE FILEPATH "Emscripten ranlib")
   set (CMAKE_AR ${EMSCRIPTEN_ROOT_PATH}/emar${EMCC_SUFFIX} CACHE FILEPATH "Emscripten ar")
   set (CMAKE_FIND_ROOT_PATH ${CMAKE_FIND_ROOT_PATH} ${EMSCRIPTEN_ROOT_PATH})
   set (CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
   set (CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
   set (CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
   set (CMAKE_SYSTEM_PREFIX_PATH ${CMAKE_FIND_ROOT_PATH})
   set (CMAKE_CXX_FLAGS_INIT "${CMAKE_CXX_FLAGS_INIT} -s WASM=1")
   set (CMAKE_EXE_LINKER_FLAGS_INIT "${CMAKE_EXE_LINKER_FLAGS_INIT} -s WASM=1")
   set (CMAKE_CXX_FLAGS_RELEASE_INIT "${CMAKE_CXX_FLAGS_RELEASE_INIT} -DNDEBUG -O3")
   set (CMAKE_EXE_LINKER_FLAGS_RELEASE_INIT "${CMAKE_EXE_LINKER_FLAGS_RELEASE_INIT} -O3 --llvm-lto 1")

   #set (CMAKE_STATIC_LIBRARY_SUFFIX ".bc")
   #set (CMAKE_C_CREATE_STATIC_LIBRARY "<CMAKE_AR> -o <TARGET> <LINK_FLAGS> <OBJECTS>")
   #set (CMAKE_CXX_CREATE_STATIC_LIBRARY "<CMAKE_AR> -o <TARGET> <LINK_FLAGS> <OBJECTS>")
   set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-warn-absolute-paths -s USE_PTHREADS=1 -s ASSERTIONS=1 --memory-init-file 0")
   set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-warn-absolute-paths -s USE_PTHREADS=1 -s ASSERTIONS=1 --memory-init-file 0")
   #set (CMAKE_EXECUTABLE_SUFFIX ".wasm.js")
endif()

include_directories (
   ${EMSCRIPTEN_ROOT_PATH}/system/include
)
endif (BUILD_EMSCRIPTEN) # BUILD_EMSCRIPTEN

if(LIVE555_MONOLITH_BUILD)
    set(live555_library_type OBJECT)
endif()

add_subdirectory(UsageEnvironment)
add_subdirectory(groupsock)
add_subdirectory(BasicUsageEnvironment)
add_subdirectory(liveMedia)

if(LIVE555_EPOLL_SUPPORT)
    add_subdirectory(EpollTaskScheduler)
endif()

if(LIVE555_BUILD_EXAMPLES)
    add_subdirectory(testProgs)
    add_subdirectory(proxyServer)
    add_subdirectory(mediaServer)
    add_subdirectory(hlsProxy)
endif()

if ( BUILD_SHARED_LIBS )
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/UsageEnvironment" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/groupsock" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/BasicUsageEnvironment" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/liveMedia" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
   if(LIVE555_EPOLL_SUPPORT)
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/EpollTaskScheduler" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
   endif()
   if(LIVE555_BUILD_EXAMPLES)
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/testProgs" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/proxyServer" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/mediaServer" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/hlsProxy" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=1")
   endif()
else ()
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/UsageEnvironment" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/groupsock" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/BasicUsageEnvironment" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
   set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/liveMedia" APPEND
      PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
   if(LIVE555_EPOLL_SUPPORT)
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/EpollTaskScheduler" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
   endif()
   if(LIVE555_BUILD_EXAMPLES)
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/testProgs" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/proxyServer" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/mediaServer" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
      set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/hlsProxy" APPEND
         PROPERTY COMPILE_DEFINITIONS "BUILD_SHARED_LIBS=0")
   endif()
endif ()


if(live555_master_project)
    set_property(GLOBAL PROPERTY USE_FOLDERS ON)
endif()

if(LIVE555_MONOLITH_BUILD)
   message(STATUS "Shared library")
   
    add_library(live555 SHARED 
        $<TARGET_OBJECTS:UsageEnvironment> 
        $<TARGET_OBJECTS:groupsock>
        $<TARGET_OBJECTS:BasicUsageEnvironment>
        $<TARGET_OBJECTS:liveMedia>
        $<TARGET_OBJECTS:EpollTaskScheduler>
    )
    target_include_directories(live555 INTERFACE
        $<BUILD_INTERFACE:${live555_SOURCE_DIR}/UsageEnvironment/include>
        $<BUILD_INTERFACE:${live555_SOURCE_DIR}/groupsock/include>
        $<BUILD_INTERFACE:${live555_SOURCE_DIR}/BasicUsageEnvironment/include>
        $<BUILD_INTERFACE:${live555_SOURCE_DIR}/liveMedia/include>
        $<BUILD_INTERFACE:${live555_SOURCE_DIR}/EpollTaskScheduler/include>
    )
    if(MSVC)
        target_link_libraries(live555 PUBLIC ws2_32)
    endif()
   set ( WS_LINK_LIBRARIES "" )
   if ( WIN32 )
      list ( APPEND WS_LINK_LIBRARIES
         ws2_32
      )
      set(LIB_EXTENSION ".dll")
   endif ()
   if(TARGET EpollTaskScheduler)
      target_link_libraries(live555 INTERFACE
          $<TARGET_OBJECTS:EpollTaskScheduler>)
      target_include_directories(live555 INTERFACE
          $<BUILD_INTERFACE:${live555_SOURCE_DIR}/EpollTaskScheduler/include>)
      add_library(live555::EpollTaskScheduler ALIAS EpollTaskScheduler)
   endif()
#   if(LIVE555_ENABLE_OPENSSL)
#     message(STATUS "find package OpenSSL.")
#     find_package(OpenSSL REQUIRED COMPONENTS Crypto SSL)
#
#     if(OpenSSL_FOUND)
#        message(STATUS "=============================================================")
#        message(STATUS "OpenSSL Notification.")
#        message(STATUS "Using OpenSSL ${OPENSSL_VERSION}")
#        message(STATUS "OpenSSL Include Dir: ${OPENSSL_INCLUDE_DIR}")
#        message(STATUS "OpenSSL Libs: ${OPENSSL_LIBRARIES}")
#        message(STATUS "=============================================================")
#        target_link_libraries(live555 PUBLIC
#           OpenSSL::SSL
#           OpenSSL::Crypto
#        )
#     endif(OpenSSL_FOUND)
#  else()
      message(STATUS "build without OpenSSL.")
      target_compile_definitions(live555 INTERFACE NO_OPENSSL=1)
#  endif()
   add_library(live555::live555 ALIAS live555)
   add_library(live555::UsageEnvironment ALIAS live555)
   add_library(live555::groupsock ALIAS live555)
   add_library(live555::BasicUsageEnvironment ALIAS live555)
   add_library(live555::liveMedia ALIAS live555)

   live555_target_version(live555
      CURRENT 1
      REVISION 0
      AGE 0
   )
   set_target_properties(live555 PROPERTIES FOLDER "live555")
else()
   message(STATUS "Static library")
    add_library(live555 INTERFACE)
    add_library(live555::live555 ALIAS live555)
    target_link_libraries(live555 INTERFACE
        UsageEnvironment
        groupsock
        BasicUsageEnvironment
        liveMedia
    )
    if(TARGET EpollTaskScheduler)
        target_link_libraries(live555 INTERFACE EpollTaskScheduler)
    endif()
endif()

if ( LIVE555_MONOLITH_BUILD )
   if(MSVC)
      target_compile_definitions(live555 INTERFACE LIVEMEDIA_API=__declspec\(dllimport\))
   else()
      target_compile_definitions(live555 INTERFACE LIVEMEDIA_API=)
   endif()
else ()
   target_compile_definitions(live555 INTERFACE LIVEMEDIA_API=)
endif ()

# Installation stuff
set(config_version_file ${CMAKE_CURRENT_BINARY_DIR}/live555ConfigVersion.cmake)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(${config_version_file}
                                 VERSION ${live555_VERSION}
                                 COMPATIBILITY ExactVersion)

if(LIVE555_BUILD_EXAMPLES)
   set(bin_files ${CMAKE_CURRENT_BINARY_DIR}/testProgs/MPEG2TransportStreamIndexer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/mikeyParse
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/openRTSP
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/playSIP
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/registerRTSPStream
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/sapWatch
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testAMRAudioStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testDVVideoStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testH264VideoStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testH264VideoToHLSSegments
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testH264VideoToTransportStream
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testH265VideoStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testH265VideoToTransportStream
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMKVSplitter
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMKVStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMP3Receiver
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMP3Streamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG1or2AudioVideoStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG1or2ProgramToTransportStream
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG1or2Splitter
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG1or2VideoReceiver
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG1or2VideoStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG2TransportReceiver
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG2TransportStreamTrickPlay
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG2TransportStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testMPEG4VideoStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testOggStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testOnDemandRTSPServer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testRTSPClient
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testRelay
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testReplicator
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/testWAVAudioStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/testProgs/vobStreamer
                 ${CMAKE_CURRENT_BINARY_DIR}/mediaServer/live555MediaServer
                 ${CMAKE_CURRENT_BINARY_DIR}/proxyServer/live555ProxyServer
                 ${CMAKE_CURRENT_BINARY_DIR}/hlsProxy/live555HLSProxy)
endif()

if(LIVE555_INSTALL)
   install(FILES ${config_version_file}
      DESTINATION lib/cmake/live555
   )

   install(TARGETS live555 EXPORT live555Target
      INCLUDES DESTINATION include/liveMedia
      RUNTIME DESTINATION bin/$<CONFIG>
      LIBRARY DESTINATION lib/$<CONFIG>
      ARCHIVE DESTINATION lib/$<CONFIG>
   )

   if(LIVE555_BUILD_EXAMPLES)
      install(FILES ${bin_files}
         DESTINATION bin
         PERMISSIONS 
            OWNER_WRITE OWNER_READ OWNER_EXECUTE
            GROUP_WRITE GROUP_READ GROUP_EXECUTE
            WORLD_WRITE WORLD_READ WORLD_EXECUTE
         COMPONENT runtime
      )
   endif()

   install(EXPORT live555Target
      NAMESPACE live555::
      DESTINATION lib/cmake/live555
      FILE live555Config.cmake
   )
endif()
