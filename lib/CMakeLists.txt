# Modified by Mikhail Gusev (gusevmihs@gmail.com) for NextGIS Build System.

configure_file(${CURL_SOURCE_DIR}/include/curl/curlbuild.h.cmake
  ${CURL_BINARY_DIR}/include/curl/curlbuild.h)
configure_file(curl_config.h.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/curl_config.h)

transform_makefile_inc("Makefile.inc" "${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
include(${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake)

list(APPEND HHEADERS
  ${CMAKE_CURRENT_BINARY_DIR}/curl_config.h
  ${CURL_BINARY_DIR}/include/curl/curlbuild.h
  )

if(MSVC)
  list(APPEND CSOURCES libcurl.rc)
endif()

# SET(CSOURCES
# #  memdebug.c -not used
# # nwlib.c - Not used
# # strtok.c - specify later
# # strtoofft.c - specify later
# )

# # if we have Kerberos 4, right now this is never on
# #OPTION(CURL_KRB4 "Use Kerberos 4" OFF)
# IF(CURL_KRB4)
# SET(CSOURCES ${CSOURCES}
# krb4.c
# security.c
# )
# ENDIF(CURL_KRB4)

# #OPTION(CURL_MALLOC_DEBUG "Debug mallocs in Curl" OFF)
# MARK_AS_ADVANCED(CURL_MALLOC_DEBUG)
# IF(CURL_MALLOC_DEBUG)
# SET(CSOURCES ${CSOURCES}
# memdebug.c
# )
# ENDIF(CURL_MALLOC_DEBUG)

# # only build compat strtoofft if we need to
# IF(NOT HAVE_STRTOLL AND NOT HAVE__STRTOI64)
# SET(CSOURCES ${CSOURCES}
# strtoofft.c
# )
# ENDIF(NOT HAVE_STRTOLL AND NOT HAVE__STRTOI64)


# The rest of the build

include_directories(${CMAKE_CURRENT_BINARY_DIR}/../include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/..)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../include)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/..)
include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})
if(USE_ARES)
  include_directories(${CARES_INCLUDE_DIR})
endif()

#if(CURL_STATICLIB)
if(NOT BUILD_SHARED_LIBS)
  # Static lib
    set(CURL_USER_DEFINED_DYNAMIC_OR_STATIC STATIC)
    if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
        set( CMAKE_CXX_FLAGS "-fpic ${CMAKE_CXX_FLAGS}" )
        set( CMAKE_C_FLAGS   "-fpic ${CMAKE_C_FLAGS}" )
    endif()
else()
  # DLL / so dynamic lib
  set(CURL_USER_DEFINED_DYNAMIC_OR_STATIC SHARED)
endif()

if(OSX_FRAMEWORK)
    set(INST_HEADER
        ${CURL_BINARY_DIR}/include/curl/curlbuild.h
        ${CURL_SOURCE_DIR}/include/curl/curl.h
        ${CURL_SOURCE_DIR}/include/curl/curlrules.h
        ${CURL_SOURCE_DIR}/include/curl/curlver.h
        ${CURL_SOURCE_DIR}/include/curl/easy.h
        ${CURL_SOURCE_DIR}/include/curl/mprintf.h
        ${CURL_SOURCE_DIR}/include/curl/multi.h
        ${CURL_SOURCE_DIR}/include/curl/stdcheaders.h
        ${CURL_SOURCE_DIR}/include/curl/typecheck-gcc.h
    )
    add_library(${LIB_NAME} SHARED ${HHEADERS} ${CSOURCES} ${INST_HEADER})
    set_target_properties(${LIB_NAME} PROPERTIES
      FRAMEWORK TRUE
      FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
      MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
      MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
      MACOSX_FRAMEWORK_IDENTIFIER org.curl.libcurl
      XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
      PUBLIC_HEADER "${INST_HEADER}"
      OUTPUT_NAME ${OUT_LIB_NAME}
    )
else()
    add_library(${LIB_NAME} ${CURL_USER_DEFINED_DYNAMIC_OR_STATIC} ${HHEADERS} ${CSOURCES})
    set_target_properties(${LIB_NAME} PROPERTIES OUTPUT_NAME ${OUT_LIB_NAME})
    if(BUILD_SHARED_LIBS AND BUILD_STATIC_LIBS)
        add_library(${LIB_NAME}static STATIC ${HHEADERS} ${CSOURCES})
        set(INSTALL_TARGETS ${INSTALL_TARGETS} ${LIB_NAME}static)
        if(WIN32)
            set_target_properties(${LIB_NAME}static PROPERTIES OUTPUT_NAME ${OUT_LIB_NAME}_i)
        else()
            set_target_properties(${LIB_NAME}static PROPERTIES OUTPUT_NAME ${OUT_LIB_NAME})
        endif()
    endif()
endif()
set(INSTALL_TARGETS ${INSTALL_TARGETS} ${LIB_NAME})


if(MSVC AND NOT BUILD_SHARED_LIBS) # CURL_STATICLIB)
    set_target_properties(${LIB_NAME} PROPERTIES STATIC_LIBRARY_FLAGS ${CMAKE_EXE_LINKER_FLAGS})
endif()

if(BUILD_SHARED_LIBS OR OSX_FRAMEWORK)
    if(WIN32)
        set (LIB_VERSION ${CURL_MAJOR_VERSION})
    else ()
        set (LIB_VERSION ${CURL_VERSION})
    endif()
    set_target_properties (${LIB_NAME} PROPERTIES VERSION ${LIB_VERSION})
    set_target_properties (${LIB_NAME} PROPERTIES SOVERSION ${CURL_MAJOR_VERSION})
endif()

set(TARGET_LINK_LIB ${TARGET_LINK_LIB} ${CURL_LIBS})
target_link_extlibraries(${LIB_NAME}) # only supported external libraries
#target_link_libraries(${LIB_NAME} ${CURL_LIBS} ${TARGET_LINK_LIB})

if(WIN32)
  add_definitions(-D_USRDLL)
endif()

set_target_properties(${LIB_NAME} PROPERTIES COMPILE_DEFINITIONS BUILDING_LIBCURL)

if(HIDES_CURL_PRIVATE_SYMBOLS)
  set_property(TARGET ${LIB_NAME} APPEND PROPERTY COMPILE_DEFINITIONS "CURL_HIDDEN_SYMBOLS")
  set_property(TARGET ${LIB_NAME} APPEND PROPERTY COMPILE_FLAGS ${CURL_CFLAG_SYMBOLS_HIDE})
endif()

# Remove the "lib" prefix since the library is already named "libcurl".
#set_target_properties(${LIB_NAME} PROPERTIES PREFIX "")
#set_target_properties(${LIB_NAME} PROPERTIES IMPORT_PREFIX "")

# if(WIN32)
#   if(BUILD_SHARED_LIBS)#NOT CURL_STATICLIB)
#     # Add "_imp" as a suffix before the extension to avoid conflicting with the statically linked "libcurl.lib"
#     set_target_properties(${LIB_NAME} PROPERTIES IMPORT_SUFFIX "_imp.lib")
#   endif()
# endif()

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(TARGETS ${LIB_NAME}
        ARCHIVE DESTINATION ${INSTALL_LIB_DIR} COMPONENT libraries
        LIBRARY DESTINATION ${INSTALL_LIB_DIR} COMPONENT libraries
        RUNTIME DESTINATION ${INSTALL_BIN_DIR} COMPONENT libraries
        FRAMEWORK DESTINATION ${INSTALL_LIB_DIR}
    )
endif()

set(EXPORT_TARGETS ${EXPORT_TARGETS} ${INSTALL_TARGETS} PARENT_SCOPE)
