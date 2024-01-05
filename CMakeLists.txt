cmake_minimum_required(VERSION 3.21)

# configs
set(LUA_VERSION
  "5.4.6"
  CACHE STRING "Lua version that should be used")

option(LUA_BUILD_AS_CXX "Complie Lua as cpp code" OFF)
option(LUA_BUILD_SHARED "Build Lua shared lib" ON)
option(LUA_BUILD_INTERPRETER "Build Lua interpreter" ON)
option(LUA_BUILD_COMPILER "Build Lua compiler" ON)
option(LUA_BUILD_VERSION_SUFFIX "Append version on output files" OFF)

if(EMSCRIPTEN)
  set(LUA_BUILD_SHARED OFF)
endif()

# project
project(Lua
  VERSION ${LUA_VERSION}
  HOMEPAGE_URL "www.lua.org")

set(LUA_PATH "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}")
set(LUA_SRC_PATH "${LUA_PATH}/src")

# download src
if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}.tar.gz")
  file(DOWNLOAD "www.lua.org/ftp/lua-${LUA_VERSION}.tar.gz"
    "${LUA_PATH}.tar.gz"
    STATUS DOWNLOAD_STATUS)

  list(GET DOWNLOAD_STATUS 0 STATUS_CODE)
  list(GET DOWNLOAD_STATUS 1 ERROR_MESSAGE)

  if(NOT ${STATUS_CODE} EQUAL 0)
    message(FATAL_ERROR "Error occurred during download: ${ERROR_MESSAGE}")
  endif()

  execute_process(
    COMMAND ${CMAKE_COMMAND} -E tar -xf
    "${LUA_PATH}.tar.gz"
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endif()

file(GLOB_RECURSE SOURCE_FILES
  "${LUA_SRC_PATH}/*.c")

# if compiled to c++ instead of c
if(LUA_BUILD_AS_CXX)
  set_source_files_properties(
    ${SOURCE_FILES}
    PROPERTIES LANGUAGE CXX)
endif()

list(REMOVE_ITEM SOURCE_FILES
  "${LUA_SRC_PATH}/luac.c"
  "${LUA_SRC_PATH}/lua.c")

if(LUA_BUILD_VERSION_SUFFIX)
  set(VERSION_SUFFIX "${PROJECT_VERSION_MAJOR}${PROJECT_VERSION_MINOR}")
endif()

# lua static lib
add_library(lua_static STATIC ${SOURCE_FILES})
add_library(lua::static ALIAS lua_static)

set_target_properties(
  lua_static PROPERTIES
  EXPORT_NAME static
  OUTPUT_NAME lua${VERSION_SUFFIX}
  POSITION_INDEPENDENT_CODE ON
)

if(APPLE)
  target_compile_definitions(lua_static PUBLIC LUA_USE_MACOSX)
elseif(UNIX)
  target_compile_definitions(lua_static PUBLIC LUA_USE_LINUX)
elseif(WIN32)
  target_compile_definitions(lua_static PRIVATE LUA_BUILD_AS_DLL)
endif()

target_include_directories(
  lua_static
  PUBLIC "$<BUILD_INTERFACE:${LUA_SRC_PATH}>")

target_compile_definitions(lua_static PUBLIC
  LUA_VERSION_MAJOR_NUM=${PROJECT_VERSION_MAJOR}
  LUA_VERSION_MINOR_NUM=${PROJECT_VERSION_MINOR}
  LUA_VERSION_PATCH_NUM=${PROJECT_VERSION_PATCH})

if(UNIX)
  target_link_libraries(lua_static PRIVATE m dl)
endif()

function(add_rc tar)
  target_sources(${tar} PRIVATE "${CMAKE_CURRENT_LIST_DIR}/resource/lua.manifest" "${CMAKE_CURRENT_LIST_DIR}/resource/lua.rc")
  target_compile_definitions(${tar} PRIVATE FILE_NAME=\"$<TARGET_FILE_NAME:${tar}>\")
endfunction()

# lua shared lib
if(LUA_BUILD_SHARED)
  add_library(lua_shared SHARED $<TARGET_OBJECTS:lua_static>)
  add_library(lua::shared ALIAS lua_shared)

  set_target_properties(
    lua_shared PROPERTIES

    # requires manual access to LUA_API
    # CXX_VISIBILITY_PRESET hidden
    # VISIBILITY_INLINES_HIDDEN YES
    VERSION "${PROJECT_VERSION}"
    SOVERSION "${PROJECT_VERSION_MAJOR}"
    EXPORT_NAME shared
    OUTPUT_NAME lua${VERSION_SUFFIX}
    ARCHIVE_OUTPUT_NAME lua_shared${VERSION_SUFFIX}
    POSITION_INDEPENDENT_CODE ON
  )

  if(WIN32)
    target_compile_definitions(lua_shared PUBLIC LUA_BUILD_AS_DLL)
  endif()

  target_link_libraries(lua_shared PUBLIC lua_static)
  add_rc(lua_shared)
endif()

# lua interpreter
if(LUA_BUILD_INTERPRETER)
  add_executable(luai "${LUA_SRC_PATH}/lua.c")
  add_executable(lua::interpreter ALIAS luai)

  set_target_properties(
    luai PROPERTIES
    VERSION "${PROJECT_VERSION}"
    EXPORT_NAME interpreter
    OUTPUT_NAME lua
    ARCHIVE_OUTPUT_NAME luai
    PDB_NAME luai
  )
  target_link_libraries(luai PRIVATE $<IF:$<TARGET_EXISTS:lua_shared>,lua_shared,lua_static>)

  if(UNIX AND NOT APPLE AND NOT EMSCRIPTEN)
    target_compile_definitions(luai PRIVATE LUA_USE_READLINE)
    target_link_libraries(luai PRIVATE readline)
  endif()

  add_rc(luai)
endif()

# lua compiler
if(LUA_BUILD_COMPILER)
  add_executable(luac "${LUA_SRC_PATH}/luac.c")
  add_executable(lua::compiler ALIAS luac)

  set_target_properties(
    luac PROPERTIES
    VERSION "${PROJECT_VERSION}"
    EXPORT_NAME compiler
    OUTPUT_NAME luac
    PDB_NAME luac
  )
  target_link_libraries(luac PRIVATE lua_static)
  add_rc(luac)
endif()

# emscripten flags
if(EMSCRIPTEN)
  if(LUA_BUILD_AS_CXX)
    target_compile_options(lua_static PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    target_link_options(lua_static PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    if(TARGET lua_shared)
      target_compile_options(lua_shared PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
      target_link_options(lua_shared PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    endif()

    if(TARGET luai)
      target_compile_options(luai PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
      target_link_options(luai PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    endif()

    if(TARGET luac)
      target_compile_options(luac PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
      target_link_options(luac PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    endif()
  else()
    target_compile_definitions(lua_static PUBLIC _longjmp=longjmp _setjmp=setjmp)
  endif()
  target_compile_definitions(lua_static PUBLIC popen=fopen) # HACK
endif()

# install
if(NOT CMAKE_SKIP_INSTALL_RULES)
  include(CMakePackageConfigHelpers)
  include(GNUInstallDirs)

  set(package lua)

  install(
    FILES "${LUA_SRC_PATH}/lauxlib.h"
    "${LUA_SRC_PATH}/lua.h"
    "${LUA_SRC_PATH}/luaconf.h"
    "${LUA_SRC_PATH}/lualib.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    COMPONENT lua_devel)

  if(NOT LUA_BUILD_AS_CXX)
    install(
      FILES
      "${LUA_SRC_PATH}/lua.hpp"
      DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
      COMPONENT lua_devel)
  endif()

  install(
    TARGETS lua_static
    EXPORT luaTargets
    ARCHIVE COMPONENT lua_devel
    INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  )

  if(TARGET lua_shared)
    install(
      TARGETS lua_shared
      EXPORT luaTargets
      RUNTIME COMPONENT lua_shared_runtime COMPONENT lua_runtime
      LIBRARY COMPONENT lua_shared_runtime COMPONENT lua_runtime NAMELINK_COMPONENT lua_devel
      ARCHIVE COMPONENT lua_devel
      INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
    )
  endif()

  if(TARGET luai)
    install(
      TARGETS luai
      EXPORT luaTargets
      RUNTIME COMPONENT lua_interpreter_runtime COMPONENT lua_runtime
    )
  endif()

  if(TARGET luac)
    install(
      TARGETS luac
      EXPORT luaTargets
      RUNTIME COMPONENT lua_compiler_runtime COMPONENT lua_runtime
    )
  endif()

  write_basic_package_version_file(
    "${package}ConfigVersion.cmake"
    COMPATIBILITY SameMajorVersion
  )

  # Allow package maintainers to freely override the path for the configs
  set(
    lua_INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/${package}"
    CACHE PATH "CMake package config location relative to the install prefix"
  )
  mark_as_advanced(lua_INSTALL_CMAKEDIR)

  file(WRITE install-config.cmake "include(\"\${CMAKE_CURRENT_LIST_DIR}/sharedTargets.cmake\")")

  install(
    FILES install-config.cmake
    DESTINATION "${lua_INSTALL_CMAKEDIR}"
    RENAME "${package}Config.cmake"
    COMPONENT lua_devel
  )

  install(
    FILES "${PROJECT_BINARY_DIR}/${package}ConfigVersion.cmake"
    DESTINATION "${lua_INSTALL_CMAKEDIR}"
    COMPONENT lua_devel
  )

  install(
    EXPORT luaTargets
    NAMESPACE lua::
    DESTINATION "${lua_INSTALL_CMAKEDIR}"
    COMPONENT lua_devel
  )
endif()
