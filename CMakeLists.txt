PROJECT(lcurl C)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8)


INCLUDE(CMakeDependentOption)
SET(CPACK_SET_DESTDIR TRUE)
INCLUDE(CPack)

# Make sure CMake can find our non-standard modules
SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/CMakeModules/" ${CMAKE_MODULE_PATH})

FIND_PACKAGE(Lua53)
FIND_PACKAGE(CURL)

INCLUDE_DIRECTORIES(${LUA_INCLUDE_DIR} ${CURL_INCLUDE_DIRS})


IF(APPLE AND IOS)
	OPTION(WANTS_BUILD_MODULE_LIBRARY "Set to ON to build loadable module." OFF)
	OPTION(WANTS_BUILD_STATIC_LIBRARY "Set to ON to build static library." ON)
ELSE()
	OPTION(WANTS_BUILD_MODULE_LIBRARY "Set to ON to build loadable module." ON)
	OPTION(WANTS_BUILD_STATIC_LIBRARY "Set to ON to build static library." OFF)
ENDIF()

IF(APPLE)
	SET(MODULE_LINK_FLAGS "-undefined dynamic_lookup")
ENDIF(APPLE)



SET(lcurl_SOURCE_FILES
		${lcurl_SOURCE_DIR}/src/l52util.c
		${lcurl_SOURCE_DIR}/src/lceasy.c
		${lcurl_SOURCE_DIR}/src/lcerror.c
		${lcurl_SOURCE_DIR}/src/lchttppost.c
		${lcurl_SOURCE_DIR}/src/lcmulti.c
		${lcurl_SOURCE_DIR}/src/lcshare.c
		${lcurl_SOURCE_DIR}/src/lcurl.c
		${lcurl_SOURCE_DIR}/src/lcutils.c

		${lcurl_SOURCE_DIR}/src/l52util.h
		${lcurl_SOURCE_DIR}/src/lceasy.h
		${lcurl_SOURCE_DIR}/src/lcerr_easy.h
		${lcurl_SOURCE_DIR}/src/lcerr_form.h
		${lcurl_SOURCE_DIR}/src/lcerr_multi.h
		${lcurl_SOURCE_DIR}/src/lcerr_share.h
		${lcurl_SOURCE_DIR}/src/lcerror.h
		${lcurl_SOURCE_DIR}/src/lcflags.h
		${lcurl_SOURCE_DIR}/src/lchttppost.h
		${lcurl_SOURCE_DIR}/src/lcinfoeasy.h
		${lcurl_SOURCE_DIR}/src/lcmulti.h
		${lcurl_SOURCE_DIR}/src/lcoptmulti.h
		${lcurl_SOURCE_DIR}/src/lcoptshare.h
		${lcurl_SOURCE_DIR}/src/lcshare.h
		${lcurl_SOURCE_DIR}/src/lcurl.h
		${lcurl_SOURCE_DIR}/src/lcutils.h
)

SET(lcurl_RESOURCE_FILES_ROOT
		${lcurl_SOURCE_DIR}/src/lua/cURL.lua
)
SET(lcurl_RESOURCE_FILES_cURL
	${lcurl_SOURCE_DIR}/src/lua/cURL/safe.lua
	${lcurl_SOURCE_DIR}/src/lua/cURL/utils.lua
)
SET(lcurl_RESOURCE_FILES_cURL_impl
	${lcurl_SOURCE_DIR}/src/lua/cURL/impl/cURL.lua
)

SET(lcurl_RESOURCE_FILES
	${lcurl_RESOURCE_FILES_ROOT}
	${lcurl_RESOURCE_FILES_cURL}
	${lcurl_RESOURCE_FILES_ROOT_cURL_impl}
)

IF(WANTS_BUILD_MODULE_LIBRARY)
	ADD_LIBRARY(lcurl_library_module MODULE 
		${lcurl_SOURCE_FILES} 
		${lcurl_RESOURCE_FILES}
	)
	TARGET_LINK_LIBRARIES(lcurl_library_module ${CURL_LIBRARIES} ${LUA_LIBRARY} ${MODULE_LINK_FLAGS})

	IF(ANDROID)
		# The problem seems to be that Android won't externalize .so modules into /data/data/packagename/lib unless
		# the module starts with a "lib" prefix, e.g. liblcurl.so instead of lcurl.so. 
		# But simply renaming the file to liblcurl.so causes internal assumptions to fail,
		# such as then needing to require('liblcurl') instead of require('lcurl').
		# But then if we do that, Lua tries to call luaopen_liblcurl instead of luaopen_lcurl.
		# So we can avoid this mess by prefixing lib to the name in the cpath.
		# REMEMBER: We must ship modules on Android with the lib prefix (liblcurl.so).
		# But users don't need to know about this in their code.
		# Also note that Java System.loadLibrary implicitly assumes the lib prefix,
		# so this is important for that too.
		SET_TARGET_PROPERTIES(lcurl_library_module PROPERTIES 
			# call the thing lcurl
			OUTPUT_NAME "lcurl"
		)
	ELSE()
		SET_TARGET_PROPERTIES(lcurl_library_module PROPERTIES 
			# make sure "lib" doesn't become a prefix
			PREFIX ""
			# call the thing lcurl
			OUTPUT_NAME "lcurl"
		)
	ENDIF()

	IF(APPLE)
		INCLUDE(XcodeDefaults)
		BLURRR_CONFIGURE_XCODE_DEFAULTS(lcurl_library_module)
	ENDIF()

	INSTALL(TARGETS
		lcurl_library_module
		#	RUNTIME DESTINATION bin
		LIBRARY DESTINATION lib
		ARCHIVE DESTINATION lib
		#	ARCHIVE DESTINATION lib/static
	)

ENDIF(WANTS_BUILD_MODULE_LIBRARY)

IF(WANTS_BUILD_STATIC_LIBRARY)
	ADD_LIBRARY(lcurl_library_static STATIC 
		${lcurl_SOURCE_FILES} 
		${lcurl_RESOURCE_FILES}
	)
#	TARGET_LINK_LIBRARIES(lcurl_library_static ${LUA_LIBRARY})
	# FIXME: Might cause a name clash in Visual Studio
	SET_TARGET_PROPERTIES(lcurl_library_static PROPERTIES OUTPUT_NAME "lcurl")
	INSTALL(TARGETS
		lcurl_library_static
		#	RUNTIME DESTINATION bin
		LIBRARY DESTINATION lib
		ARCHIVE DESTINATION lib
		#	ARCHIVE DESTINATION lib/static
	)

	IF(APPLE)
		INCLUDE(XcodeDefaults)
		BLURRR_CONFIGURE_XCODE_DEFAULTS(lcurl_library_static)
	ENDIF()


ENDIF(WANTS_BUILD_STATIC_LIBRARY)

IF(LUA_INCLUDE_DIR AND EXISTS "${LUA_INCLUDE_DIR}/lua.h")
	file(STRINGS "${LUA_INCLUDE_DIR}/lua.h" LUA_VERSION_MAJOR_LINE REGEX "^#define[ \t]+LUA_VERSION_MAJOR[ \t]+\"[0-9]+\"$")
	file(STRINGS "${LUA_INCLUDE_DIR}/lua.h" LUA_VERSION_MINOR_LINE REGEX "^#define[ \t]+LUA_VERSION_MINOR[ \t]+\"[0-9]+\"$")
	string(REGEX REPLACE "^#define[ \t]+LUA_VERSION_MAJOR[ \t]+\"([0-9]+)\"$" "\\1" LUA_VERSION_MAJOR "${LUA_VERSION_MAJOR_LINE}")
	string(REGEX REPLACE "^#define[ \t]+LUA_VERSION_MINOR[ \t]+\"([0-9]+)\"$" "\\1" LUA_VERSION_MINOR "${LUA_VERSION_MINOR_LINE}")

	SET(LUAPACKAGE_CDIR "lib/lua/${LUA_VERSION_MAJOR}.${LUA_VERSION_MINOR}" CACHE STRING "Path for Lua packaged platform specific things.")
	SET(LUAPACKAGE_LDIR "share/lua/${LUA_VERSION_MAJOR}.${LUA_VERSION_MINOR}" CACHE STRING "Path for Lua packages platform independent things.")
ENDIF()
INSTALL(FILES ${lcurl_RESOURCE_FILES_ROOT} DESTINATION ${LUAPACKAGE_LDIR})
INSTALL(FILES ${lcurl_RESOURCE_FILES_cURL} DESTINATION ${LUAPACKAGE_LDIR}/cURL)
INSTALL(FILES ${lcurl_RESOURCE_FILES_cURL_impl} DESTINATION ${LUAPACKAGE_LDIR}/cURL/impl)




# For uninstall (needs cmake_uninstall.cmake.in in the top-level directory)
CONFIGURE_FILE(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  IMMEDIATE @ONLY)
ADD_CUSTOM_TARGET(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")



