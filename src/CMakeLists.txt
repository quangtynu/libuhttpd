
add_definitions(-O -Wall -Werror --std=gnu99 -D_GNU_SOURCE)

# The version number.
set(UHTTPD_VERSION_MAJOR 3)
set(UHTTPD_VERSION_MINOR 0)
set(UHTTPD_VERSION_PATCH 2)

# Check the third party Libraries
find_package(Libev REQUIRED)

include_directories(${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/buffer ${CMAKE_CURRENT_BINARY_DIR} ${LIBEV_INCLUDE_DIR})

set(EXTRA_LIBS ${LIBEV_LIBRARY} dl m)
set(SOURCE_FILES uhttpd.c log.c connection.c buffer/buffer.c http_parser.c ssl.c)

set(UHTTPD_SSL_SUPPORT_CONFIG 1)
option(UHTTPD_SSL_SUPPORT "SSL support" ON)

option(UHTTPD_USE_OPENSSL "Force select OpenSSL" OFF)
option(UHTTPD_USE_WOLFSSL "Force select WolfSSL(CyaSSL)" OFF)
option(UHTTPD_USE_MBEDTLS "Force select MbedTLS(PolarSSL)" OFF)

set(SSL_NAME OFF)
set(UHTTPD_HAVE_OPENSSL_CONFIG 0)
set(UHTTPD_HAVE_WOLFSSL_CONFIG 0)
set(UHTTPD_HAVE_MBEDTLS_CONFIG 0)

if(NOT UHTTPD_SSL_SUPPORT)
    set(UHTTPD_SSL_SUPPORT_CONFIG 0)
else()
    find_package(OpenSSL)
    find_package(WolfSSL)
    find_package(MbedTLS)

	if(UHTTPD_USE_OPENSSL)
		if (NOT OPENSSL_FOUND)
			set(UHTTPD_SSL_SUPPORT OFF)
			message(WARNING "Force select OpenSSL, but not found it")
		endif()
	elseif(UHTTPD_USE_WOLFSSL)
		if (NOT WOLFSSL_FOUND)
			set(UHTTPD_SSL_SUPPORT OFF)
			message(WARNING "Force select WolfSSL(CyaSSL), but not found it")
		endif()
	elseif(UHTTPD_USE_MBEDTLS)
		if (NOT MBEDTLS_FOUND)
			set(UHTTPD_SSL_SUPPORT OFF)
			message(WARNING "Force select MbedTLS(PolarSSL), but not found it")
		endif()
	elseif(OPENSSL_FOUND)
		set(UHTTPD_USE_OPENSSL ON)
	elseif(WOLFSSL_FOUND)
		set(UHTTPD_USE_WOLFSSL ON)
	elseif(MBEDTLS_FOUND)
		set(UHTTPD_USE_MBEDTLS ON)
	else()
		set(UHTTPD_SSL_SUPPORT OFF)
		message(WARNING "No available SSL libraries found")
	endif()

	if(UHTTPD_USE_OPENSSL)
		set(SSL_NAME "OpenSSL")
		set(SSL_INC ${OPENSSL_INCLUDE_DIR})
		set(SSL_LIB ${OPENSSL_LIBRARIES})
		set(UHTTPD_HAVE_OPENSSL_CONFIG 1)
	elseif(UHTTPD_USE_WOLFSSL)
		set(SSL_NAME "WolfSSL(CyaSSL)")
		set(SSL_INC ${WOLFSSL_INCLUDE_DIR})
		set(SSL_LIB ${WOLFSSL_LIBRARIES})
		set(UHTTPD_HAVE_WOLFSSL_CONFIG 1)
	elseif(UHTTPD_USE_MBEDTLS)
		set(SSL_NAME "MbedTLS(PolarSSL)")
		set(SSL_INC ${MBEDTLS_INCLUDE_DIR})
		set(SSL_LIB ${MBEDTLS_LIBRARIES})
		set(UHTTPD_HAVE_MBEDTLS_CONFIG 1)
	endif()

    if(UHTTPD_SSL_SUPPORT)
	    include_directories(${SSL_INC})
        list(APPEND EXTRA_LIBS ${SSL_LIB})
        message(STATUS "Select ${SSL_NAME} as the SSL backend")
    else()
	    set(SSL_NAME OFF)
	    set(UHTTPD_SSL_SUPPORT_CONFIG 0)
    endif()
endif()

add_library(uhttpd SHARED ${SOURCE_FILES})
set_target_properties(uhttpd PROPERTIES VERSION ${UHTTPD_VERSION_MAJOR}.${UHTTPD_VERSION_MINOR}.${UHTTPD_VERSION_PATCH})
target_link_libraries(uhttpd ${EXTRA_LIBS})

add_library(uhttpd_s STATIC ${SOURCE_FILES})
set_target_properties(uhttpd_s PROPERTIES OUTPUT_NAME uhttpd)
target_link_libraries(uhttpd_s ${EXTRA_LIBS})

# configure a header file to pass some of the CMake settings to the source code
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

install(
    FILES
        uhttpd.h
        connection.h
        log.h
        buffer/buffer.h
        http_parser.h
        ${CMAKE_CURRENT_BINARY_DIR}/config.h
    DESTINATION
        include/uhttpd
)

install(
    TARGETS uhttpd uhttpd_s
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

message(STATUS "UHTTPD_VERSION: ${UHTTPD_VERSION_MAJOR}.${UHTTPD_VERSION_MINOR}.${UHTTPD_VERSION_PATCH}")
message(STATUS "UHTTPD_SSL_SUPPORT: ${SSL_NAME}")

