cmake_minimum_required(VERSION 3.16)
project(pvr.sublime CXX C)

# Built without Kodi's addon build system: the client needs only the header-only
# kodi-dev-kit, pugixml and zlib. KODI_DEVKIT points at xbmc/addons/kodi-dev-kit
# of the Kodi the boxes run (22.0 RC1, xbmc 4665ec22a4).
set(ADDON_VERSION "22.1.0" CACHE STRING "add-on version")
set(KODI_DEVKIT "" CACHE PATH "xbmc/addons/kodi-dev-kit")
set(PUGIXML_DIR "" CACHE PATH "pugixml source tree")
set(ZLIB_ROOT_DIR "" CACHE PATH "static zlib install prefix")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(SOURCES src/PVRLinkData.cpp
            src/tvlink/CatchupController.cpp
            src/tvlink/Channels.cpp
            src/tvlink/ChannelGroups.cpp
            src/tvlink/Epg.cpp
            src/tvlink/PlaylistLoader.cpp
            src/tvlink/Settings.cpp
            src/tvlink/StreamManager.cpp
            src/tvlink/data/Channel.cpp
            src/tvlink/data/ChannelEpg.cpp
            src/tvlink/data/ChannelGroup.cpp
            src/tvlink/data/EpgEntry.cpp
            src/tvlink/data/EpgGenre.cpp
            src/tvlink/utilities/FileUtils.cpp
            src/tvlink/utilities/Logger.cpp
            src/tvlink/utilities/StreamUtils.cpp
            src/tvlink/utilities/WebUtils.cpp
            ${PUGIXML_DIR}/src/pugixml.cpp)

add_library(pvr.sublime SHARED ${SOURCES})
target_include_directories(pvr.sublime PRIVATE ${KODI_DEVKIT}/include ${PUGIXML_DIR}/src)
if(ZLIB_ROOT_DIR)
  target_include_directories(pvr.sublime PRIVATE ${ZLIB_ROOT_DIR}/include)
endif()
target_compile_definitions(pvr.sublime PRIVATE IPTV_VERSION=${ADDON_VERSION})
option(KODI21 "build for Kodi 21 (Omega) PVR API" OFF)
if(KODI21)
  target_compile_definitions(pvr.sublime PRIVATE PVR_KODI21)
endif()
# Платформа Kodi (addon.xml: library_<платформа> и <platform>) и имя библиотеки - как у
# официальных бинарных дополнений: Linux pvr.sublime.so.<версия>, Windows pvr.sublime.dll,
# Android libpvr.sublime.so, macOS pvr.sublime.dylib (universal: x86_64 + arm64).
if(WIN32)
  target_compile_options(pvr.sublime PRIVATE /O2 /utf-8 /EHsc)
  target_compile_definitions(pvr.sublime PRIVATE NOMINMAX _CRT_SECURE_NO_WARNINGS WIN32_LEAN_AND_MEAN)
  target_link_libraries(pvr.sublime PRIVATE ${ZLIB_ROOT_DIR}/lib/zlibstatic.lib)
  set_target_properties(pvr.sublime PROPERTIES PREFIX "" OUTPUT_NAME "pvr.sublime" SUFFIX ".dll")
  set(PLATFORM_LIB windx)
  set(PLATFORM_TAG "windx windows windows-x86_64")
  set(LIBRARY_FILENAME pvr.sublime.dll)
elseif(ANDROID)
  target_compile_options(pvr.sublime PRIVATE -O2 -fvisibility=hidden)
  target_link_libraries(pvr.sublime PRIVATE z -Wl,--no-undefined)
  set_target_properties(pvr.sublime PROPERTIES PREFIX "lib" OUTPUT_NAME "pvr.sublime" SUFFIX ".so")
  set(PLATFORM_LIB android)
  if(ANDROID_ABI STREQUAL "armeabi-v7a")
    set(PLATFORM_TAG "android-armv7")
  else()
    set(PLATFORM_TAG "android-aarch64")
  endif()
  set(LIBRARY_FILENAME libpvr.sublime.so)
elseif(APPLE)
  target_compile_options(pvr.sublime PRIVATE -O2 -fvisibility=hidden)
  target_link_libraries(pvr.sublime PRIVATE z)
  set_target_properties(pvr.sublime PROPERTIES PREFIX "" OUTPUT_NAME "pvr.sublime" SUFFIX ".dylib")
  set(PLATFORM_LIB osx)
  set(PLATFORM_TAG "osx64 osx-x86_64 osxarm64 osx-arm64")
  set(LIBRARY_FILENAME pvr.sublime.dylib)
else()
  target_compile_options(pvr.sublime PRIVATE -O2 -fvisibility=hidden)
  # libstdc++/libgcc inside the .so: the box's are newer, but nothing to match is safer
  target_link_libraries(pvr.sublime PRIVATE ${ZLIB_ROOT_DIR}/lib/libz.a -static-libstdc++ -static-libgcc -Wl,--no-undefined)
  set_target_properties(pvr.sublime PROPERTIES PREFIX "" OUTPUT_NAME "pvr.sublime" SUFFIX ".so.${ADDON_VERSION}")
  set(PLATFORM_LIB linux)
  set(PLATFORM_TAG linux)
  set(LIBRARY_FILENAME pvr.sublime.so.${ADDON_VERSION})
endif()

# addon.xml: the same imports Kodi's own addon build writes, read from versions.h
file(READ ${KODI_DEVKIT}/include/kodi/versions.h VERSIONS_H)
set(ADDON_DEPENDS "")
foreach(api GLOBAL:MAIN:kodi.binary.global.main GLOBAL:GENERAL:kodi.binary.global.general
            GLOBAL:FILESYSTEM:kodi.binary.global.filesystem GLOBAL:TOOLS:kodi.binary.global.tools
            INSTANCE:PVR:kodi.binary.instance.pvr)
  string(REPLACE ":" ";" parts ${api})
  list(GET parts 0 kind)
  list(GET parts 1 name)
  list(GET parts 2 id)
  string(REGEX MATCH "#define ADDON_${kind}_VERSION_${name} +\"([0-9.]+)\"" _ "${VERSIONS_H}")
  set(ver ${CMAKE_MATCH_1})
  string(REGEX MATCH "#define ADDON_${kind}_VERSION_${name}_MIN +\"([0-9.]+)\"" _ "${VERSIONS_H}")
  set(min ${CMAKE_MATCH_1})
  if(NOT ver OR NOT min)
    message(FATAL_ERROR "no version for ${id} in versions.h")
  endif()
  string(APPEND ADDON_DEPENDS "
<import addon=\"${id}\" minversion=\"${min}\" version=\"${ver}\"/>")
endforeach()
configure_file(pvr.sublime/addon.xml.in ${CMAKE_BINARY_DIR}/addon/pvr.sublime/addon.xml @ONLY)
