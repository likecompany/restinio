cmake_minimum_required(VERSION 3.5)

SET(CMAKE_CXX_STANDARD 14)
SET(CMAKE_CXX_STANDARD_REQUIRED ON)

option(RESTINIO_INSTALL "Generate the install target." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_TEST "Build the tests." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_SAMPLE "Build samples." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_INSTALL_SAMPLES "Build install samples." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_BENCH "Build the test target." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_INSTALL_BENCHES "Build install samples." ${RESTINIO_MASTER_PROJECT})
option(RESTINIO_FIND_DEPS "Use `find_package()` for including RESTinio dependencies." OFF)
option(RESTINIO_FMT_HEADER_ONLY "Include fmt library as header-only." OFF)

option(RESTINIO_ALLOW_SOBJECTIZER "Allow usage of SObjectizer" ${RESTINIO_MASTER_PROJECT})

option(RESTINIO_USE_EXTERNAL_SOBJECTIZER "Allow add_subdirectory(so_5) in the main RESTinio CMakeLists.txt" OFF)

# Since v.0.6.5 a user can force usage of already installed http-parser
# library.
option(RESTINIO_USE_EXTERNAL_HTTP_PARSER "Use already installed http-parser library" ON)

# Since v.0.6.7 a user can force usage of already installed nonstd
# libraries.
option(RESTINIO_USE_EXTERNAL_EXPECTED_LITE
	"Use already installed expected-lite library" OFF)
option(RESTINIO_USE_EXTERNAL_OPTIONAL_LITE
	"Use already installed optional-lite library" OFF)
option(RESTINIO_USE_EXTERNAL_STRING_VIEW_LITE
	"Use already installed string-view-lite library" OFF)
option(RESTINIO_USE_EXTERNAL_VARIANT_LITE
	"Use already installed variant-lite library" OFF)

SET(RESTINIO_USE_BOOST_ASIO "none" CACHE STRING "Use boost version of ASIO")
SET(RESTINIO_USE_BOOST_ASIO_VALUES "none;static;shared")

set_property(CACHE RESTINIO_USE_BOOST_ASIO PROPERTY STRINGS ${RESTINIO_USE_BOOST_ASIO_VALUES})

project(RESTinioRootProject CXX)

IF (RESTINIO_USE_EXTERNAL_SOBJECTIZER AND NOT RESTINIO_ALLOW_SOBJECTIZER)
	message(FATAL_ERROR "RESTINIO_USE_EXTERNAL_SOBJECTIZER can't be set without RESTINIO_ALLOW_SOBJECTIZER")
ENDIF()

IF (RESTINIO_ALLOW_SOBJECTIZER)
	SET(RESTINIO_SOBJECTIZER_ENABLED ON)
ENDIF()

IF (RESTINIO_MASTER_PROJECT)
	# RESTinio is a main project (not subproject)
	# Do regular stuff here.

	list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")
	include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/link_threads_if_necessary.cmake")
	include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/link_atomic_if_necessary.cmake")

	# ------------------------------------------------------------------------------
	# RESTinio dependencies:
	# ASIO or Boost::ASIO
	IF ( RESTINIO_USE_BOOST_ASIO MATCHES "none" )
		SET( RESTINIO_STAND_ALONE_ASIO_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/asio/include )
		SET( RESTINIO_STAND_ALONE_ASIO_DEFINES -DASIO_STANDALONE -DASIO_HAS_STD_CHRONO -DASIO_DISABLE_STD_STRING_VIEW)

		# Try to find boost for boost regex.
		SET(Boost_USE_MULTITHREADED ON)
		find_package(Boost COMPONENTS regex)
		if ( Boost_FOUND )
			message(STATUS "Found Boost for regex.")
		endif ()
	ELSE ()
		SET(Boost_USE_MULTITHREADED ON)
		IF ( RESTINIO_USE_BOOST_ASIO MATCHES "shared" )
			SET(Boost_USE_STATIC_LIBS OFF)
		ELSE ()
			SET(Boost_USE_STATIC_LIBS ON)
		ENDIF ()

		find_package(Boost 1.66.0 REQUIRED COMPONENTS system regex)
	ENDIF ()

	IF (RESTINIO_FIND_DEPS)
		# Require necessary packages.

		# Catch2
		IF (RESTINIO_TEST)
			find_package(Catch2 CONFIG REQUIRED)
		ENDIF ()

		# HTTP parser
		IF (RESTINIO_USE_EXTERNAL_HTTP_PARSER)
			find_package(http-parser REQUIRED)
		ELSE ()
			find_package(unofficial-http-parser CONFIG REQUIRED)
		ENDIF ()

		# fmtlib
		find_package(fmt REQUIRED)
	ELSE ()
		# Add necessary dependecies.

		include_directories("${CMAKE_SOURCE_DIR}/clara")

		# HTTP parser
		IF (RESTINIO_USE_EXTERNAL_HTTP_PARSER)
			find_package(http-parser REQUIRED)
		ELSE ()
			add_subdirectory(nodejs/http_parser)
		ENDIF ()

		# fmtlib
		find_package(fmt REQUIRED)
	ENDIF ()

	# If RESTINIO_USE_EXTERNAL_*_LITE is defined we should
	# search that libs regardless of RESTINIO_FIND_DEPS option.
	IF (RESTINIO_USE_EXTERNAL_EXPECTED_LITE)
		find_package(expected-lite REQUIRED)
	ENDIF()
	IF (RESTINIO_USE_EXTERNAL_OPTIONAL_LITE)
		find_package(optional-lite REQUIRED)
	ENDIF()
	IF (RESTINIO_USE_EXTERNAL_STRING_VIEW_LITE)
		find_package(string-view-lite REQUIRED)
	ENDIF()
	IF (RESTINIO_USE_EXTERNAL_VARIANT_LITE)
		find_package(variant-lite REQUIRED)
	ENDIF()

	# ------------------------------------------------------------------------------

	# ------------------------------------------------------------------------------
	# Zlib
	find_package(ZLIB)

	IF (NOT ZLIB_FOUND)
		add_subdirectory(restinio/third_party/zlib)
		message( STATUS "USE OWN ZLIB SOURCES")
		SET(ZLIB_LIBRARIES zlibstatic)
		SET(ZLIB_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/restinio/third_party/zlib)
	ENDIF ()

	message( STATUS "ZLIB_LIBRARIES='" ${ZLIB_LIBRARIES} "'" )
	message( STATUS "ZLIB_INCLUDE_DIRS='" ${ZLIB_INCLUDE_DIRS} "'" )
ENDIF (RESTINIO_MASTER_PROJECT)

# RESTinio itself.
add_subdirectory(restinio)
