set(EXE_NAME curl)

if(USE_MANUAL)
  find_package(Perl REQUIRED)
  # Use the C locale to ensure that only ASCII characters appear in the
  # embedded text. NROFF and MANOPT are set in the parent CMakeLists.txt
  add_custom_command(
    OUTPUT tool_hugehelp.c
    COMMAND echo "#include \"tool_setup.h\"" > tool_hugehelp.c
    COMMAND echo "#ifndef HAVE_LIBZ" >> tool_hugehelp.c
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_SOURCE_DIR}/docs/curl.1" |
            "${PERL}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl"
            "${CURL_SOURCE_DIR}/docs/MANUAL" >> tool_hugehelp.c
    COMMAND echo "#else" >> tool_hugehelp.c
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_SOURCE_DIR}/docs/curl.1" |
            "${PERL}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl" -c
            "${CURL_SOURCE_DIR}/docs/MANUAL" >> tool_hugehelp.c
    COMMAND echo "#endif /* HAVE_LIBZ */" >> tool_hugehelp.c
    DEPENDS
      "${CURL_SOURCE_DIR}/docs/MANUAL"
      "${CURL_SOURCE_DIR}/docs/curl.1"
      "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl"
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)
else()
  add_custom_command(
    OUTPUT tool_hugehelp.c
    COMMAND echo "/* built-in manual is disabled, blank function */" > tool_hugehelp.c
    COMMAND echo "#include \"tool_hugehelp.h\"" >> tool_hugehelp.c
    COMMAND echo "void hugehelp(void) {}" >> tool_hugehelp.c
    DEPENDS
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)
endif()

transform_makefile_inc("Makefile.inc" "${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
include(${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake)

if(MSVC)
  list(APPEND CURL_SOURCE curl.rc)
endif()

add_executable(
  ${EXE_NAME}
  ${curl_SOURCES}
  )

source_group("cURLX source files" FILES ${CURLX_CFILES})
source_group("cURL source files" FILES ${CURL_CFILES})
source_group("cURL header files" FILES ${CURL_HFILES})

include_directories(
  ${CURL_SOURCE_DIR}/lib        # To be able to reach "curl_setup_once.h"
  ${CURL_BINARY_DIR}/lib        # To be able to reach "curl_config.h"
  ${CURL_BINARY_DIR}/include    # To be able to reach "curl/curlbuild.h"
  # This is needed as tool_hugehelp.c is generated in the binary dir
  ${CURL_SOURCE_DIR}/src        # To be able to reach "tool_hugehelp.h"
  )

#Build cURL executable
target_link_libraries( ${EXE_NAME} libcurl ${CURL_LIBS})

################################################################################

#SET_TARGET_PROPERTIES(${EXE_NAME} ARCHIVE_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} RUNTIME_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} LIBRARY_OUTPUT_DIRECTORY "blah blah blah")

# Add the postfix to the executable since it is not added automatically as for modules and shared libraries
set_target_properties(${EXE_NAME} PROPERTIES
  DEBUG_POSTFIX "${CMAKE_DEBUG_POSTFIX}")

#INCLUDE(ModuleInstall OPTIONAL)

install(TARGETS ${EXE_NAME} DESTINATION bin)