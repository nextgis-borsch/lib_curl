set(EXE_NAME curl)

if(USE_MANUAL)
  # Use the C locale to ensure that only ASCII characters appear in the
  # embedded text. NROFF and MANOPT are set in the parent CMakeLists.txt
  add_custom_command(
    OUTPUT tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#include \"tool_setup.h\"" > tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#ifndef HAVE_LIBZ" >> tool_hugehelp.c
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_BINARY_DIR}/docs/curl.1" |
            "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl"
            "${CURL_SOURCE_DIR}/docs/MANUAL" >> tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#else" >> tool_hugehelp.c
    COMMAND env LC_ALL=C "${NROFF}" ${NROFF_MANOPT}
            "${CURL_BINARY_DIR}/docs/curl.1" |
            "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl" -c
            "${CURL_SOURCE_DIR}/docs/MANUAL" >> tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#endif /* HAVE_LIBZ */" >> tool_hugehelp.c
    DEPENDS
      "${CURL_SOURCE_DIR}/docs/MANUAL"
      generate-curl.1
      "${CURL_BINARY_DIR}/docs/curl.1"
      "${CMAKE_CURRENT_SOURCE_DIR}/mkhelp.pl"
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)
else()
  add_custom_command(
    OUTPUT tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "/* built-in manual is disabled, blank function */" > tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "#include \"tool_hugehelp.h\"" >> tool_hugehelp.c
    COMMAND ${CMAKE_COMMAND} -E echo "void hugehelp(void) {}" >> tool_hugehelp.c
    DEPENDS
      "${CMAKE_CURRENT_SOURCE_DIR}/tool_hugehelp.h"
    VERBATIM)

endif()

transform_makefile_inc("Makefile.inc" "${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake")
include(${CMAKE_CURRENT_BINARY_DIR}/Makefile.inc.cmake)

if(MSVC)
  list(APPEND CURL_FILES curl.rc)
endif()

# CURL_FILES comes from Makefile.inc
add_executable(
  ${EXE_NAME}
  ${CURL_FILES}
  )

if(CURL_HAS_LTO)
  set_target_properties(${EXE_NAME} PROPERTIES
    INTERPROCEDURAL_OPTIMIZATION_RELEASE TRUE
    INTERPROCEDURAL_OPTIMIZATION_RELWITHDEBINFO TRUE)
endif()

if(ENABLE_UNICODE AND MINGW)
  target_link_libraries(${EXE_NAME} -municode)
  # GCC doesn't know about wmain
  set_source_files_properties(tool_main.c PROPERTIES COMPILE_FLAGS "-Wno-missing-prototypes -Wno-missing-declarations")
endif()

if(OSX_FRAMEWORK)
    set_target_properties(${EXE_NAME} PROPERTIES INSTALL_RPATH "@executable_path/../../Library/Frameworks")
endif()

source_group("curlX source files" FILES ${CURLX_CFILES})
source_group("curl source files" FILES ${CURL_CFILES})
source_group("curl header files" FILES ${CURL_HFILES})

include_directories(
  ${CURL_SOURCE_DIR}/lib        # To be able to reach "curl_setup_once.h"
  ${CURL_BINARY_DIR}/lib        # To be able to reach "curl_config.h"
  ${CURL_BINARY_DIR}/include    # To be able to reach "curl/curl.h"
  # This is needed as tool_hugehelp.c is generated in the binary dir
  ${CURL_SOURCE_DIR}/src        # To be able to reach "tool_hugehelp.h"
  )

#Build curl executable
target_link_libraries(${EXE_NAME} libcurl ${CURL_LIBS})

################################################################################

#SET_TARGET_PROPERTIES(${EXE_NAME} ARCHIVE_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} RUNTIME_OUTPUT_DIRECTORY "blah blah blah")
#SET_TARGET_PROPERTIES(${EXE_NAME} LIBRARY_OUTPUT_DIRECTORY "blah blah blah")

# Add the postfix to the executable since it is not added automatically as for modules and shared libraries
set_target_properties(${EXE_NAME} PROPERTIES
  DEBUG_POSTFIX "${CMAKE_DEBUG_POSTFIX}")

#INCLUDE(ModuleInstall OPTIONAL)

if(NOT SKIP_INSTALL_EXECUTABLES AND NOT SKIP_INSTALL_ALL )
    install(TARGETS ${EXE_NAME}
        RUNTIME DESTINATION ${INSTALL_BIN_DIR} COMPONENT applications
        FRAMEWORK DESTINATION ${INSTALL_BIN_DIR}
    )
endif()

if(BUILD_TESTING AND WIN32)
    file(GLOB_RECURSE IMPORTED_WIN_DLLS ${CMAKE_BINARY_DIR}/third-party/*.dll)
    foreach(IMPORTED_WIN_DLL ${IMPORTED_WIN_DLLS})
        add_custom_command(TARGET curl
                            POST_BUILD
                            COMMAND ${CMAKE_COMMAND} -E copy ${IMPORTED_WIN_DLL} $<TARGET_FILE_DIR:curl>
                            COMMENT "Copy ${IMPORTED_WIN_DLL} to $<TARGET_FILE_DIR:curl>"
                        )
    endforeach()
    add_custom_command(TARGET curl
                        POST_BUILD
                        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:libcurl> $<TARGET_FILE_DIR:curl>
                        COMMENT "Copy $<TARGET_FILE:libcurl> to $<TARGET_FILE_DIR:curl>"
                    )
endif()
