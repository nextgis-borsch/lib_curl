# Modified by Mikhail Gusev (gusevmihs@gmail.com) for NextGIS Build System.
# Modified by Dmitry Baryshnikov (dmitry.baryshnikov@nextgis.com) to fit NextGIS Borsch v2
# Copyright (C) 2015-2018, NextGIS <info@nextgis.com>

set(LIB_NAME libcurl)

if(BUILD_SHARED_LIBS OR OSX_FRAMEWORK)
  set(CURL_STATICLIB OFF)
else()
  set(CURL_STATICLIB ON)
endif()

# Use:
# * CURL_STATICLIB
configure_file(curl_config.h.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/curl_config.h)

transform_makefile_inc("Makefile.inc" "${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
include(${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake)

list(APPEND HHEADERS
  ${CMAKE_CURRENT_BINARY_DIR}/curl_config.h
  )

if(MSVC)
  list(APPEND CSOURCES libcurl.rc)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4127")
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

if(OSX_FRAMEWORK)
    set(OUT_LIB_NAME curl)
    file(GLOB INST_HEADER ${CURL_SOURCE_DIR}/include/curl/*.h)
    set(INST_HEADER ${INST_HEADER}
        ${CMAKE_CURRENT_BINARY_DIR}/curl_config.h
    )
    set_property(SOURCE ${INST_HEADER} PROPERTY MACOSX_PACKAGE_LOCATION "Headers/curl")

    add_library(${LIB_NAME} SHARED ${HHEADERS} ${CSOURCES} ${INST_HEADER})
    set_target_properties(${LIB_NAME} PROPERTIES
      FRAMEWORK TRUE
      FRAMEWORK_VERSION ${FRAMEWORK_VERSION}
      MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${VERSION}
      MACOSX_FRAMEWORK_BUNDLE_VERSION ${VERSION}
      MACOSX_FRAMEWORK_IDENTIFIER org.curl.libcurl
      XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
      # PUBLIC_HEADER "${INST_HEADER}"
      OUTPUT_NAME ${OUT_LIB_NAME}
    )
else()
    add_library(${LIB_NAME} ${HHEADERS} ${CSOURCES})
endif()
set(EXPORT_TARGETS ${EXPORT_TARGETS} ${LIB_NAME})

if(MSVC AND CURL_STATICLIB)
  set_target_properties(${LIB_NAME} PROPERTIES STATIC_LIBRARY_FLAGS ${CMAKE_EXE_LINKER_FLAGS})
endif()

set(TARGET_LINK_LIB ${TARGET_LINK_LIB} ${CURL_LIBS})
target_link_extlibraries(${LIB_NAME}) # only supported external libraries

if(WIN32)
  add_definitions( -D_USRDLL )
endif()

set_target_properties(${LIB_NAME} PROPERTIES COMPILE_DEFINITIONS BUILDING_LIBCURL)

if(HIDES_CURL_PRIVATE_SYMBOLS)
  set_property(TARGET ${LIB_NAME} APPEND PROPERTY COMPILE_DEFINITIONS "CURL_HIDDEN_SYMBOLS")
  set_property(TARGET ${LIB_NAME} APPEND PROPERTY COMPILE_FLAGS ${CURL_CFLAG_SYMBOLS_HIDE})
endif()

# Remove the "lib" prefix since the library is already named "libcurl".
set_target_properties(${LIB_NAME} PROPERTIES PREFIX "")
set_target_properties(${LIB_NAME} PROPERTIES IMPORT_PREFIX "")

if(WIN32)
  if(BUILD_SHARED_LIBS)
    # Add "_imp" as a suffix before the extension to avoid conflicting with the statically linked "libcurl.lib"
    set_target_properties(${LIB_NAME} PROPERTIES IMPORT_SUFFIX "_imp.lib")
  endif()
endif()

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(TARGETS ${LIB_NAME}
        EXPORT ${PACKAGE_UPPER_NAME}Targets
        RUNTIME DESTINATION ${INSTALL_BIN_DIR}
        LIBRARY DESTINATION ${INSTALL_LIB_DIR}
        ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
        INCLUDES DESTINATION ${INSTALL_INC_DIR}
        FRAMEWORK DESTINATION ${INSTALL_LIB_DIR} )
endif()

# Add path to includes to build-tree export
target_include_directories(${EXPORT_TARGETS} PUBLIC
  $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
)

# Add all targets to the build-tree export set
export(TARGETS ${EXPORT_TARGETS}
    FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

# Create the CURLConfig.cmake file
configure_file(${PROJECT_SOURCE_DIR}/CMake/PackageConfig.cmake.in
    ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    # Install the <Package>Config.cmake
    install(FILES
      ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
      DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

    # Install the export set for use with the install-tree
    install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()
