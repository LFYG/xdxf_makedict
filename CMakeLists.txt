PROJECT(makedict)

# Older versions have a different signature for CMAKE_MINIMUM_REQUIRED,
# check it manually just to make sure
IF (${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} LESS 2.6)
  MESSAGE(FATAL_ERROR "${PROJECT_NAME} requires at least CMake v2.6."
		" You are running v${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}."
		" Please upgrade." )
ENDIF(${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION} LESS 2.6)

# If we get this far, use the modern signature.  This will also cause newer
# CMake versions to try to be backwards-compatible with the desired version
CMAKE_MINIMUM_REQUIRED(VERSION 2.6 FATAL_ERROR)
cmake_policy(VERSION 2.6)

# Disable deprecation warnings for standard C and STL functions in VS2005
# and later
IF(CMAKE_COMPILER_2005)
  ADD_DEFINITIONS(-D_CRT_SECURE_NO_DEPRECATE -D_CRT_NONSTDC_NO_DEPRECATE)
  ADD_DEFINITIONS(-D_SCL_SECURE_NO_DEPRECATE)
ENDIF(CMAKE_COMPILER_2005)

SET(GLIB2_REQ "'glib-2.0 >= 2.6.0'")
#INCLUDE("${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindPackageHandleStandardArgs.cmake")
# where to look first for cmake modules, before ${CMAKE_ROOT}/Modules/ is checked
SET(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
INCLUDE("${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindGLIB2.cmake")

IF (NOT GLIB2_FOUND)
	MESSAGE(FATAL_ERROR "makedict require ${GLIB2_REQ}, "
		"make sure that you install it")
ENDIF(NOT GLIB2_FOUND)

SET(ZLIB_FIND_REQUIRED True)
INCLUDE(FindZLIB)

SET(GETTEXT_FIND_REQUIRED True)
INCLUDE(FindGettext)

SET(ICONV_FIND_REQUIRED True)
INCLUDE("${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindIconv.cmake")

SET(LIBINTL_FIND_REQUIRED True)
INCLUDE("${CMAKE_CURRENT_SOURCE_DIR}/cmake/FindLibintl.cmake")

SET(EXPAT_FIND_REQUIRED True)
SET(EXPAT_FIND_QUIETLY True)
INCLUDE(FindEXPAT)

INCLUDE(CheckFunctionExists)
Check_Function_Exists(mmap HAVE_MMAP)

INCLUDE(CheckIncludeFile)
CHECK_INCLUDE_FILE(locale.h HAVE_LOCALE_H)

CONFIGURE_FILE(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake
	${CMAKE_CURRENT_BINARY_DIR}/config.h)

INCLUDE_DIRECTORIES(
	${ICONV_INCLUDE_DIR}
	${GETTEXT_INCLUDE_DIR}
	${LIBINTL_INCLUDE_DIR}
	${GLIB2_INCLUDE_DIRS}
	${GLIBCONFIG_INCLUDE_DIR}
	${EXPAT_INCLUDE_DIRS}
	${ZLIB_INCLUDE_DIR}
	${CMAKE_CURRENT_BINARY_DIR}
)

#TODO: may be special flag here?
IF (NOT WIN32)
ADD_CUSTOM_COMMAND(OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/src/dsl_ipa.cpp
	COMMAND "/bin/sh" ARGS "${CMAKE_CURRENT_SOURCE_DIR}/src/ipa_tbl_to_cpp.sh"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/dsl_ipa.cpp"
	"${CMAKE_CURRENT_SOURCE_DIR}/src/dsl_ipa_tbl.txt"
	MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/src/dsl_ipa_tbl.txt
)
ENDIF (NOT WIN32)

SET(CODECS src/mueller7_parser.py src/apresyan.py)
SET(CODECS_LIB src/mdparser.py)

SET(makedict_SRCS
	src/compiler.hpp
	src/makedict.cpp
	src/file.cpp
	src/parser.cpp
	src/generator.cpp
	src/process.cpp
	src/connector.cpp
	src/log.cpp
	src/utils.cpp
	src/xml.cpp
	src/charset_conv.cpp
	src/dictd_generator.cpp
	src/dictd_parser.cpp
	src/dsl_ipa.cpp
	src/dsl_parser.cpp
	src/dummy_generator.cpp
	src/dummy_parser.cpp
	src/lang_tbl.cpp
	src/lang_tbl_auto.cpp
	src/mapfile.cpp
	src/normalize_tags.cpp
	src/sdict_parser.cpp
	src/stardict_generator.cpp
	src/stardict_parser.cpp
	src/xdxf_generator.cpp
	src/xdxf_parser.cpp
)

SET(makedict_HDRS
	src/file.hpp
	src/parser.hpp
	src/generator.hpp
	src/process.hpp
	src/connector.hpp
	src/log.hpp
	src/utils.hpp
	src/xml.hpp
	src/charset_conv.hpp
	src/dsl_ipa.hpp
	src/lang_tbl.hpp
	src/mapfile.hpp
	src/normalize_tags.hpp
	src/repository.hpp
	src/resource.hpp
)

ADD_EXECUTABLE(makedict
	${makedict_SRCS}
	${makedict_HDRS}
)
TARGET_LINK_LIBRARIES(makedict
  ${ZLIB_LIBRARIES}
  ${GETTEXT_LIBRARIES}
  ${LIBINTL_LIBRARIES}
  ${ICONV_LIBRARIES} 
  ${GLIB2_LIBRARIES}
  ${EXPAT_LIBRARIES}  
)

INSTALL(TARGETS makedict RUNTIME DESTINATION bin)
INSTALL(FILES docs/makedict.1 DESTINATION share/man/man1)
INSTALL(FILES ${CODECS} DESTINATION lib/makedict-codecs
	PERMISSIONS OWNER_READ OWNER_EXECUTE GROUP_EXECUTE GROUP_READ
	WORLD_EXECUTE WORLD_READ)

INSTALL(FILES ${CODECS_LIB} DESTINATION lib/makedict-codecs
	PERMISSIONS OWNER_READ GROUP_READ
	WORLD_READ)

#
# Packing stuff
#
SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Converter from any dictionary format to any")
SET(CPACK_PACKAGE_VENDOR "Evgeniy Dushistov <dushistov@mail.ru>")
SET(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
SET(CPACK_PACKAGE_VERSION_MAJOR "0")
SET(CPACK_PACKAGE_VERSION_MINOR "4")
SET(CPACK_PACKAGE_VERSION_PATCH "1-beta1")
#SET(CPACK_PACKAGE_EXECUTABLES "lua" "lua")

#SET_TARGET_PROPERTIES(makedict PROPERTIES VERSION "${CPACK_PACKAGE_VERSION_MAJOR}")
#GET_TARGET_PROPERTY(makedict_VERSION makedict VERSION)
SET(makedict_VERSION
	"${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")
SET(CODECSDIR "${CMAKE_INSTALL_PREFIX}/lib/makedict-codecs")

ADD_DEFINITIONS(-DVERSION="${makedict_VERSION}"
	-DCODECSDIR="${CODECSDIR}"
	-DDUMMY_FORMAT_NAME="dummy"
	-DHAVE_CONFIG_H)

IF (DEBUG_IPA)
  ADD_DEFINITIONS(-DDEBUG_IPA)
ENDIF (DEBUG_IPA)



INCLUDE(CPack)
OPTION(BUILD_TESTS "Enable automatic testing" False)
IF (BUILD_TESTS)
	#todo: check that all tools necessary for sh scripts exists
	FIND_PROGRAM(SHELL_CMD NAMES sh bash
          DOC "Shell scripts interpretator command")
        MESSAGE(STATUS "Build tests")
        ENABLE_TESTING()# Force "make test" to works
        ADD_SUBDIRECTORY(tests)
ENDIF (BUILD_TESTS)
