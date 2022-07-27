cmake_minimum_required(VERSION 3.21)

# configs
set(LUA_VERSION
    "5.4.4"
    CACHE STRING "Lua version that should be used")

option(LUA_CPP "Complie Lua as cpp code" OFF)

# project
project(Lua
        VERSION ${LUA_VERSION}
        HOMEPAGE_URL "www.lua.org")

# download src
if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}.tar.gz")
  file(DOWNLOAD "www.lua.org/ftp/lua-${LUA_VERSION}.tar.gz"
       "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}.tar.gz"
       STATUS DOWNLOAD_STATUS)

  list(GET DOWNLOAD_STATUS 0 STATUS_CODE)
  list(GET DOWNLOAD_STATUS 1 ERROR_MESSAGE)
  if(NOT ${STATUS_CODE} EQUAL 0)
    message(FATAL_ERROR "Error occurred during download: ${ERROR_MESSAGE}")
  endif()
  execute_process(
    COMMAND ${CMAKE_COMMAND} -E tar -xf
            "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}.tar.gz"
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
endif()

file(GLOB_RECURSE SOURCE_FILES
     "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}/src/*.c")

list(REMOVE_ITEM SOURCE_FILES
     "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}/src/luac.c"
     "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}/src/lua.c")

add_library(lua-obj OBJECT ${SOURCE_FILES})

set_property(TARGET lua-obj PROPERTY POSITION_INDEPENDENT_CODE ON)

if(UNIX AND NOT APPLE)
  target_compile_definitions(lua-obj PUBLIC LUA_USE_LINUX)
elseif(WIN32)
  target_compile_definitions(lua-obj PUBLIC LUA_BUILD_AS_DLL)
endif()

target_include_directories(
  lua-obj
  PUBLIC "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>/lua-${LUA_VERSION}/src")

target_compile_definitions(lua-obj PUBLIC 
  LUA_VERSION_MAJOR_NUM=${PROJECT_VERSION_MAJOR}
  LUA_VERSION_MINOR_NUM=${PROJECT_VERSION_MINOR}
  LUA_VERSION_PATCH_NUM=${PROJECT_VERSION_PATCH})

if(UNIX AND NOT APPLE)
  target_link_libraries(lua-obj PRIVATE m dl)
endif()

# lua lib
if(EMSCRIPTEN)
  add_library(lua)
else()
  add_library(lua SHARED)
endif()
add_library(lua::lua ALIAS lua)

function(add_rc tar)
  target_sources(${tar} PRIVATE "${CMAKE_CURRENT_LIST_DIR}/resource/lua.manifest" "${CMAKE_CURRENT_LIST_DIR}/resource/lua.rc")
  target_compile_definitions(${tar} PRIVATE FILE_NAME=\"$<TARGET_FILE_NAME:${tar}>\")
endfunction()

target_link_libraries(lua PUBLIC lua-obj)
add_rc(lua)

# lua interpreter
add_executable(luai)
add_executable(lua::luai ALIAS luai)

target_sources(luai PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}/src/lua.c")
target_link_libraries(luai PRIVATE lua)
if(UNIX AND NOT APPLE AND NOT EMSCRIPTEN)
  target_compile_definitions(luai PRIVATE LUA_USE_READLINE)
  target_link_libraries(luai PRIVATE readline)
endif()
add_rc(luai)

set_target_properties(luai PROPERTIES OUTPUT_NAME lua PDB_NAME luai)

# lua compiler
add_executable(luac)
add_executable(lua::luac ALIAS luac)

target_sources(luac PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}/src/luac.c")
target_link_libraries(luac PRIVATE lua-obj)
add_rc(luac)

# if compiled to c++ instead of c
if(LUA_CPP)
  if(NOT CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    target_compile_options(lua-obj PRIVATE -xc++)
    target_compile_options(luai PRIVATE -xc++)
    target_compile_options(luac PRIVATE -xc++)
  endif()
  set_source_files_properties(
     ${SOURCE_FILES}
    "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}/src/luac.c"
    "${CMAKE_CURRENT_BINARY_DIR}/lua-${LUA_VERSION}/src/lua.c"
    PROPERTIES LANGUAGE CXX)
endif()

# emscripten flags
if(EMSCRIPTEN)
  if(LUA_CPP)
    target_compile_options(lua-obj
                           PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    target_link_options(lua-obj PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    target_compile_options(luac PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    target_link_options(luac PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    target_compile_options(luai
                           PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
    target_link_options(luai PRIVATE "SHELL:-s DISABLE_EXCEPTION_CATCHING=0")
  else()
    target_compile_definitions(lua-obj PUBLIC _longjmp=longjmp _setjmp=setjmp)
  endif()
  target_compile_definitions(lua-obj PUBLIC popen=fopen) # HACK
endif()

# install
install(TARGETS lua luai luac RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
                                         COMPONENT lua)
