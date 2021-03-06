
# check cmake requirements
cmake_minimum_required (VERSION 3.0)

if(COMMAND cmake_policy)
	cmake_policy(SET CMP0003 NEW)

	# IF(CMAKE_VERSION VERSION_GREATER 2.8.10)
	# 	cmake_policy(SET CMP0020 NEW)	# qt warnings
	# ENDIF()
	# IF(CMAKE_VERSION VERSION_GREATER 2.8.12.9)
	# 	cmake_policy(SET CMP0043 NEW)	# COMPILE_DEFINITIONS_<CONFIG> warnings
	# ENDIF()
endif(COMMAND cmake_policy)

######################## begin of project
project(nomacs)

# needed for soname
set(NOMACS_VERSION_MAJOR 3)
set(NOMACS_VERSION_MINOR 14)
set(NOMACS_VERSION_PATCH 0)
set(NOMACS_VERSION ${NOMACS_VERSION_MAJOR}.${NOMACS_VERSION_MINOR})
set(NOMACS_FULL_VERSION ${NOMACS_VERSION}.${NOMACS_VERSION_PATCH})

if(CMAKE_CL_64)
	SET(NMC_ARCHITECTURE "x64")
else()
	SET(NMC_ARCHITECTURE "x86")
endif()

add_definitions(-DNOMACS_VERSION="${NOMACS_VERSION}")

set(BINARY_NAME ${CMAKE_PROJECT_NAME})
set(NOMACS_BUILD_DIRECTORY ${CMAKE_BINARY_DIR})

# include macros needed
include("cmake/Utils.cmake")

set(QT5_MIN_VERSION 5.2.1)

# different compile options
option(ENABLE_OPENCV "Compile with Opencv (needed for RAW and TIFF)" ON)
option(ENABLE_RAW "Compile with raw images support (libraw)" ON)
option(ENABLE_TIFF "Compile with multi-layer tiff" ON)
option(ENABLE_QT_DEBUG "Disable Qt Debug Messages" ON)
option(ENABLE_QUAZIP "Compile with QuaZip (allows opening .zip files)" ON)
option(ENABLE_INCREMENTER "Run Build Incrementer" OFF)
option(ENABLE_TRANSLATIONS "Compile Translations" OFF)
option(ENABLE_READ_BUILD "Build nomacs for READ" OFF)
option(ENABLE_PLUGINS "Compile nomacs with plugin support" ON)
option(ENABLE_HEIF "Compile nomacs with HEIF support" OFF)
option(ENABLE_CODE_COV "Run Code Coverage tests" OFF)

# Codecov
if (ENABLE_CODE_COV AND CMAKE_COMPILER_IS_GNUCXX)
    include("cmake/CodeCoverage.cmake")
    setup_target_for_coverage(${PROJECT_NAME}_coverage tests coverage)
endif()

if(APPLE)
	set(ENABLE_QUAZIP OFF)
	message(STATUS "QuaZip disabled because you are using a mac")
endif()

if(${ENABLE_READ_BUILD}) # cannot be incorporated into if one line above
	# set output path when building for READ
	if(MSVC)
		SET(CMAKE_LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR})
		SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/$<CONFIGURATION>)
		SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/$<CONFIGURATION>)
		message(STATUS "READ build turned on...")
	else()
		set(NOMACS_BUILD_DIRECTORY ${CMAKE_BINARY_DIR}/READ)
		SET(CMAKE_LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/READ)
		SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/READ)
		SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/READ)
	endif()
endif()

# load paths from the user file if exists
if (EXISTS ${CMAKE_SOURCE_DIR}/CMakeUserPaths.cmake)
	include(${CMAKE_SOURCE_DIR}/CMakeUserPaths.cmake)
elseif(MSVC)
	message(WARNING "Could not find CMakeUserPaths.cmake - please use this file to specify your install directories (see CMakeUserPathsGit.cmake)")
endif()

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
elseif(NOT MSVC)
	message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

if (ENABLE_CODE_COV AND CMAKE_COMPILER_IS_GNUCXX)
	# Codecov
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -pedantic -pthread -g -O0 -fprofile-arcs -ftest-coverage")
endif()

# find Qt
NMC_FINDQT()

if (!ENABLE_QT_DEBUG)
	message (STATUS "disabling qt debug messages")
	add_definitions(-DQT_NO_DEBUG_OUTPUT)
endif()

if(MSVC)
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Win.cmake)
elseif(APPLE)
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Mac.cmake)
elseif(UNIX)
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Unix.cmake)
else()
	message(STATUS "build system unkown ... fallback to unix")
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/Unix.cmake)
endif()

file(GLOB NOMACS_EXE_SOURCES "src/*.cpp")
file(GLOB NOMACS_EXE_HEADERS "src/*.h")

# gui
file(GLOB GUI_SOURCES "src/DkGui/*.cpp")
file(GLOB GUI_HEADERS "src/DkGui/*.h")

# core
file(GLOB CORE_SOURCES "src/DkCore/*.cpp")
file(GLOB CORE_HEADERS "src/DkCore/*.h")

# gather information for building
include_directories (
	${EXIV2_INCLUDE_DIRS}
	${LIBRAW_INCLUDE_DIRECTORY}
	${CMAKE_BINARY_DIR}
	${CMAKE_CURRENT_SOURCE_DIR}/src
	${CMAKE_CURRENT_SOURCE_DIR}/src/DkCore
	${CMAKE_CURRENT_SOURCE_DIR}/src/DkGui
	${TIFF_INCLUDE_DIR}
	${TIFF_CONFIG_DIR}
	${QUAZIP_INCLUDE_DIR}
	${QUAZIP_ZLIB_INCLUDE_DIR}
	${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/libqpsd	# needed for linux psd hack
	${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/drif
)

if (APPLE) # todo: somehow add this to Mac.cmake or MacBuildTarget.cmake
	SET (NOMACS_SOURCES ${NOMACS_SOURCES} macosx/nomacs.icns)
endif (APPLE)

IF(NOT ENABLE_PLUGINS)
	LIST(REMOVE_ITEM GUI_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/src/DkPluginManager.cpp)
	LIST(REMOVE_ITEM GUI_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/src/DkPluginManager.h)
ENDIF(NOT ENABLE_PLUGINS)


set(NOMACS_FORMS src/nomacs.ui)
set(NOMACS_RESOURCES src/nomacs.qrc)

file(GLOB NOMACS_TRANSLATIONS "translations/*.ts")

QT5_ADD_RESOURCES(NOMACS_RCC ${NOMACS_RESOURCES})

if (${ENABLE_TRANSLATIONS})
	QT5_ADD_TRANSLATION(NOMACS_QM ${NOMACS_TRANSLATIONS})
endif()

if(MSVC)
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/WinBuildTarget.cmake)
elseif(APPLE)
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/MacBuildTarget.cmake)
elseif(UNIX)
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/UnixBuildTarget.cmake)
else()
	message(STATUS "build system unkown ... fallback to unix")
	include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/UnixBuildTarget.cmake)
endif()

NMC_INSTALL()

NMC_COPY_FILES()

#debug for printing out all variables
# get_cmake_property(_variableNames VARIABLES)
# foreach (_variableName ${_variableNames})
#     message(STATUS "${_variableName}=${${_variableName}}")
# endforeach()

if(UNIX AND ENABLE_PLUGINS)

	if (NOT EXISTS ${PLUGINS_DIR})
		set(PLUGINS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/plugins")
		message(STATUS "setting plugins dir")
	else ()
		message(STATUS "${PLUGINS_DIR} is defined")
	endif()

	if (NOT EXISTS ${PLUGINS_DIR} AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/plugins")
		set(PLUGINS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/plugins")
	elseif (NOT EXISTS ${PLUGINS_DIR} AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/../../nomacs-plugins/src")
		# set the snapcraft/flathub dir
		set(PLUGINS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../../nomacs-plugins/src")
	endif ()

	if(EXISTS "${PLUGINS_DIR}")
		message(STATUS "using plugin directory: ${PLUGINS_DIR}")
		set(NOMACS_FOUND true)
		set(NOMACS_VARS_ALREADY_SET true)
		add_subdirectory(${PLUGINS_DIR} "${CMAKE_CURRENT_BINARY_DIR}/plugins")
	else()
		message(STATUS "I failed to locate plugins in: ${PLUGINS_DIR}")
		message(WARNING "plugins directory not found, not building plugins. You have to check out the nomacs-plugins git to the 'plugins' folder if you want to build them")
		SET(ENABLE_PLUGINS OFF)
	endif()
endif()

# get rid of VSCode warning (when using MSBuild)
# see: https://stackoverflow.com/questions/36451368/get-rid-of-cmake-warning-manually-specified-variables-were-not-used-by-the-proj/36452112
set(ignoreMe "${CMAKE_EXPORT_COMPILE_COMMANDS}")

# status
MESSAGE(STATUS "")
MESSAGE(STATUS "----------------------------------------------------------------------------------")
MESSAGE(STATUS " ${PROJECT_NAME} - Image Lounge ${NOMACS_VERSION}  <https://nomacs.org>")
MESSAGE(STATUS "")

IF(OpenCV_FOUND)
    MESSAGE(STATUS " nomacs will be compiled with OPENCV support .................. YES")
ELSE()
    MESSAGE(STATUS " nomacs will be compiled with OPENCV support .................. NO")
ENDIF()

IF(LIBRAW_FOUND)
    MESSAGE(STATUS " nomacs will be compiled with LIBRAW support .................. YES")
ELSE()
    MESSAGE(STATUS " nomacs will be compiled with LIBRAW support .................. NO")
ENDIF()

IF(ENABLE_TIFF)
    MESSAGE(STATUS " nomacs will be compiled with extended TIFF support ........... YES")
ELSE()
    MESSAGE(STATUS " nomacs will be compiled with extended TIFF support ........... NO")
ENDIF()

IF(ENABLE_PLUGINS)
    MESSAGE(STATUS " nomacs will be compiled with plugin support .................. YES")
ELSE()
    MESSAGE(STATUS " nomacs will be compiled with plugin support .................. NO")
ENDIF()

IF(libheif_FOUND)
    MESSAGE(STATUS " nomacs will be compiled with HEIF support .................... YES")
ELSE()
    MESSAGE(STATUS " nomacs will be compiled with HEIF support .................... NO")
ENDIF()

IF(ENABLE_TRANSLATIONS)
    MESSAGE(STATUS " nomacs will be compiled with Translations .................... YES")
ELSE()
    MESSAGE(STATUS " nomacs will be compiled with Translations .................... NO")
ENDIF()

IF(ENABLE_QUAZIP)
    MESSAGE(STATUS " nomacs will be compiled with QuaZip support .................. YES")
ELSE()
    MESSAGE(STATUS " nomacs will be compiled with QuaZip support .................. NO")
ENDIF()

IF(USE_SYSTEM_QUAZIP)
    MESSAGE(STATUS " nomacs will use system QuaZip ................................ YES")
ELSE()
    MESSAGE(STATUS " nomacs will use system QuaZip ................................ NO")
ENDIF()

IF(USE_SYSTEM_LIBQPSD)
    MESSAGE(STATUS " nomacs will use system QPSD .................................. YES")
ELSE()
    MESSAGE(STATUS " nomacs will use system QPSD .................................. NO")
ENDIF()

MESSAGE(STATUS "----------------------------------------------------------------------------------")
