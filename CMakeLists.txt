cmake_minimum_required(VERSION 3.5)

project (luv C CXX ASM)

set(LUV_VERSION_MAJOR 1)
set(LUV_VERSION_MINOR 48)
set(LUV_VERSION_PATCH 0)
set(LUV_VERSION ${LUV_VERSION_MAJOR}.${LUV_VERSION_MINOR}.${LUV_VERSION_PATCH})

if(NOT ${CMAKE_VERSION} VERSION_LESS "3.5.0")
    # generate compilation database used by C code intelligence tools
  set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
endif()

option(BUILD_MODULE "Build as module" ON)
option(BUILD_STATIC_LIBS "Build static library" OFF)
option(BUILD_SHARED_LIBS "Build shared library" OFF)
option(WITH_SHARED_LIBUV "Link to a shared libuv library instead of static linking" OFF)

if (MINGW)
  add_definitions(-D_WIN32_WINNT=0x0600)
endif (MINGW)

if (WIN32)
  # replace /MD to /MT to avoid link msvcr*.dll
  # this needs to be before add_subdirectory calls so that they inherit the modified flags
  set(CompilerFlags
    CMAKE_C_FLAGS
    CMAKE_C_FLAGS_DEBUG
    CMAKE_C_FLAGS_MINSIZEREL
    CMAKE_C_FLAGS_RELWITHDEBINFO
    CMAKE_C_FLAGS_RELEASE)
  foreach(CompilerFlag ${CompilerFlags})
    string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
  endforeach()
endif ()

if (NOT LUA_BUILD_TYPE)
  set(LUA_BUILD_TYPE "Static"
    CACHE STRING "Build Luau as static, dynamic libary, or use system one" FORCE)
  set_property(CACHE LUA_BUILD_TYPE
    PROPERTY STRINGS  "Static;Dynamic;System")
endif (NOT LUA_BUILD_TYPE)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")

if (WITH_SHARED_LIBUV)
  find_package(Libuv REQUIRED)
  include_directories(${LIBUV_INCLUDE_DIR})
else (WITH_SHARED_LIBUV)
  include_directories(deps/libuv/include)
  add_subdirectory(deps/libuv EXCLUDE_FROM_ALL)
  if (BUILD_MODULE)
    add_definitions( -DBUILDING_UV_SHARED )
  endif (BUILD_MODULE)
  set(LIBUV_LIBRARIES uv_a)
  set_target_properties(uv_a PROPERTIES COMPILE_FLAGS "-fPIC")
endif (WITH_SHARED_LIBUV)

if (LUA_BUILD_TYPE STREQUAL System)
	if (USE_LUAJIT)
		# We only link the libs on Windows, so find_package fully succeeding
		# is only required on Windows
		if (WIN32 OR CYGWIN)
			find_package(LuaJIT REQUIRED)
			link_directories(${LUAJIT_LIBRARIES})
		else()
			find_package(LuaJIT)
		endif()
		if(NOT LUAJIT_INCLUDE_DIR)
			message( FATAL_ERROR "Failed to find LuaJIT headers. Variable `LUAJIT_INCLUDE_DIR' expected to be defined.")
		endif()
		include_directories(${LUAJIT_INCLUDE_DIR})
	else (USE_LUAJIT)
		# We only link the libs on Windows, so find_package fully succeeding
		# is only required on Windows
		if (WIN32 OR CYGWIN)
			find_package(Lua REQUIRED)
		else()
			find_package(Lua)
		endif()
		if(NOT LUA_INCLUDE_DIR)
			message( FATAL_ERROR "Failed to find Lua headers. Variable `LUA_INCLUDE_DIR' expected to be defined.")
		endif()
		include_directories(${LUA_INCLUDE_DIR})
	endif (USE_LUAJIT)
else (LUA_BUILD_TYPE STREQUAL System)
	set(LUAU_STATIC_CRT ON)
	set(LUAU_BUILD_CLI OFF)
	set(LUAU_BUILD_TESTS OFF)
	set(LUAU_EXTERN_C ON)
	add_subdirectory(deps/luau EXCLUDE_FROM_ALL)
	#include_directories(${Luau_SOURCE_DIR})
	#set(LUA_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${LUA_INCLUDE_DIR})
endif (LUA_BUILD_TYPE STREQUAL System)

if (BUILD_MODULE)
  add_library(luv MODULE src/luv.c)
  set_target_properties(luv PROPERTIES PREFIX "")
  list(APPEND ACTIVE_TARGETS "luv")
endif (BUILD_MODULE)
if (BUILD_STATIC_LIBS)
  add_library(libluv_a STATIC src/luv.c)
  set_target_properties(libluv_a PROPERTIES OUTPUT_NAME luv)
  list(APPEND ACTIVE_TARGETS "libluv_a")
endif (BUILD_STATIC_LIBS)
if (BUILD_SHARED_LIBS)
  add_library(libluv SHARED src/luv.c)
  set_target_properties(libluv
    PROPERTIES VERSION ${LUV_VERSION} SOVERSION ${LUV_VERSION_MAJOR} OUTPUT_NAME luv)
  if(APPLE)
    set_target_properties(libluv
      PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
  endif(APPLE)
  list(APPEND ACTIVE_TARGETS "libluv")
endif (BUILD_SHARED_LIBS)

if(APPLE)
  set(CMAKE_SHARED_MODULE_CREATE_C_FLAGS
    "${CMAKE_SHARED_MODULE_CREATE_C_FLAGS} -flat_namespace -undefined suppress"
  )
  # execute_process(COMMAND which luajit OUTPUT_VARIABLE LUAJIT)
  # set(CMAKE_SHARED_MODULE_CREATE_C_FLAGS
  #   "${CMAKE_SHARED_MODULE_CREATE_C_FLAGS} -bundle_loader ${LUAJIT}"
  # )
endif()

foreach(target Luau.VM Luau.Compiler)
  get_target_property(defs ${target} INTERFACE_COMPILE_DEFINITIONS)
  list(FILTER defs EXCLUDE REGEX [[LUA.*_API]])
  set_target_properties(${target} PROPERTIES INTERFACE_COMPILE_DEFINITIONS "${defs}")
endforeach()

foreach(TARGET_NAME ${ACTIVE_TARGETS})
  if(WIN32 OR CYGWIN)
		target_link_libraries(${TARGET_NAME} ${LIBUV_LIBRARIES} Luau.VM Luau.Compiler)
    #if (LUA)
    #else (LUA)
    #  if (USE_LUAJIT)
    #    target_link_libraries(${TARGET_NAME} ${LIBUV_LIBRARIES} ${LUAJIT_LIBRARIES})
    #  else (USE_LUAJIT)
    #    if (LUA_BUILD_TYPE STREQUAL System)
    #        target_link_libraries(${TARGET_NAME} ${LIBUV_LIBRARIES} ${LUA_LIBRARIES})
    #    else (LUA_BUILD_TYPE STREQUAL System)
    #        target_link_libraries(${TARGET_NAME} ${LIBUV_LIBRARIES} lualib)
    #    endif (LUA_BUILD_TYPE STREQUAL System)
    #  endif (USE_LUAJIT)
    #endif (LUA)
  elseif("${CMAKE_SYSTEM_NAME}" MATCHES "Linux")
    target_link_libraries(${TARGET_NAME} ${LIBUV_LIBRARIES} rt)
  else()
    target_link_libraries(${TARGET_NAME} ${LIBUV_LIBRARIES})
  endif()
endforeach()

if (BUILD_MODULE)
	if (WIN32)
		set(MODULE_INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib")
	else (WIN32)
		set(MODULE_INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib/lua/${LUA_VERSION_MAJOR}.${LUA_VERSION_MINOR}")
	endif (WIN32)
endif (BUILD_MODULE)
if (BUILD_STATIC_LIBS)
	set(STATICLIBS_INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib"
		CACHE PATH "Installation directory for static libraries")
endif (BUILD_STATIC_LIBS)
if (BUILD_SHARED_LIBS)
	set(SHAREDLIBS_INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/bin"
		CACHE PATH "Installation directory for runtime components")
	set(SHAREDLIBS_INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib"
		CACHE PATH "Installation directory for shared libraries")
endif (BUILD_SHARED_LIBS)

# header install paths are LuaRocks-agnostic, so just use CMAKE_INSTALL_PREFIX regardless
if (BUILD_STATIC_LIBS)
  set(STATICLIBS_INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include/luv"
    CACHE PATH "Installation directory for headers")
endif (BUILD_STATIC_LIBS)
if (BUILD_SHARED_LIBS)
  set(SHAREDLIBS_INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include/luv"
    CACHE PATH "Installation directory for headers")
endif (BUILD_SHARED_LIBS)

if (CMAKE_INSTALL_PREFIX)
  if (BUILD_MODULE)
    install(TARGETS luv
      ARCHIVE DESTINATION "${MODULE_INSTALL_LIB_DIR}"
      LIBRARY DESTINATION "${MODULE_INSTALL_LIB_DIR}"
    )
  endif (BUILD_MODULE)
  if (BUILD_STATIC_LIBS)
    install(TARGETS libluv_a
      ARCHIVE DESTINATION "${STATICLIBS_INSTALL_LIB_DIR}"
      LIBRARY DESTINATION "${STATICLIBS_INSTALL_LIB_DIR}"
    )
    install(
      FILES src/luv.h src/util.h src/lhandle.h src/lreq.h
      DESTINATION "${STATICLIBS_INSTALL_INC_DIR}"
    )
  endif (BUILD_STATIC_LIBS)
  if (BUILD_SHARED_LIBS)
    install(TARGETS libluv
      RUNTIME DESTINATION "${SHAREDLIBS_INSTALL_BIN_DIR}"
      ARCHIVE DESTINATION "${SHAREDLIBS_INSTALL_LIB_DIR}"
      LIBRARY DESTINATION "${SHAREDLIBS_INSTALL_LIB_DIR}"
    )
    if(UNIX OR MINGW)
      set(prefix ${CMAKE_INSTALL_PREFIX})
      set(includedir ${SHAREDLIBS_INSTALL_INC_DIR})
      set(libdir ${SHAREDLIBS_INSTALL_LIB_DIR})
      configure_file(libluv.pc.in ${CMAKE_CURRENT_BINARY_DIR}/libluv.pc @ONLY)
      install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libluv.pc
        DESTINATION ${SHAREDLIBS_INSTALL_LIB_DIR}/pkgconfig)
    endif()
    install(
      FILES src/luv.h src/util.h src/lhandle.h src/lreq.h
      DESTINATION "${SHAREDLIBS_INSTALL_INC_DIR}"
    )
  endif (BUILD_SHARED_LIBS)
endif (CMAKE_INSTALL_PREFIX)
