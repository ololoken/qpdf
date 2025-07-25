set(MAIN_CXX_PROGRAMS
  qpdf
  fix-qdf
  pdf_from_scratch
  test_char_sign
  test_driver
  test_large_file
  test_many_nulls
  test_parsedoffset
  test_pdf_doc_encoding
  test_pdf_unicode
  test_renumber
  test_shell_glob
  test_tokenizer
  test_unicode_filenames
  test_xref)
set(MAIN_C_PROGRAMS
  qpdf-ctest
  qpdfjob-ctest)

foreach(PROG ${MAIN_CXX_PROGRAMS})
  add_executable(${PROG} ${PROG}.cc)
  target_link_libraries(${PROG} libqpdf)
endforeach()
foreach(PROG ${MAIN_C_PROGRAMS})
  add_executable(${PROG} ${PROG}.c)
  target_link_libraries(${PROG} libqpdf)
  set_property(TARGET ${PROG} PROPERTY LINKER_LANGUAGE CXX)
endforeach()

set(needs_private_headers
  test_large_file
  test_driver
  qpdf-ctest
  qpdfjob-ctest
  )
foreach(TARGET ${needs_private_headers})
  target_include_directories(${TARGET} PRIVATE ${CMAKE_SOURCE_DIR}/libqpdf)
endforeach()

foreach(B qpdf test_unicode_filenames fix-qdf test_shell_glob)
  if(WINDOWS_WMAIN_COMPILE)
    target_compile_options(${B} PRIVATE ${WINDOWS_WMAIN_COMPILE})
  endif()
  if(WINDOWS_WMAIN_LINK)
    target_link_options(${B} PRIVATE ${WINDOWS_WMAIN_LINK})
  endif()
endforeach()

add_test(
  NAME qpdf
  COMMAND ${RUN_QTEST}
  --top ${qpdf_SOURCE_DIR}
  --bin $<TARGET_FILE_DIR:qpdf>
  --bin $<TARGET_FILE_DIR:libqpdf>  # for Windows to find DLL
  --bin $<TARGET_FILE_DIR:qpdf-test-compare>
  --code ${qpdf_SOURCE_DIR}/qpdf
  --color ${QTEST_COLOR}
  --show-on-failure ${SHOW_FAILED_TEST_OUTPUT}
  --tc "${qpdf_SOURCE_DIR}/qpdf/*.cc"
  --tc "${qpdf_SOURCE_DIR}/qpdf/*.c"
  --tc "${qpdf_SOURCE_DIR}/libqpdf/*.cc")

install(TARGETS qpdf fix-qdf
  TYPE RUNTIME
  COMPONENT ${COMPONENT_CLI})

if (CMAKE_SYSTEM_NAME MATCHES "Emscripten")
  set(USE_FLAGS "--use-port=libjpeg")

  set_target_properties(qpdf PROPERTIES
          COMPILE_FLAGS "${USE_FLAGS}"
          LINK_FLAGS "${USE_FLAGS} \
              -sALLOW_MEMORY_GROWTH \
              -sENVIRONMENT=web \
              -sMODULARIZE \
              -sEXPORT_ALL -sEXPORTED_RUNTIME_METHODS=callMain,addRunDependency,removeRunDependency \
              -sEXIT_RUNTIME"
          SUFFIX ".js"
  )
endif ()

if(MINGW)
  # For MSVC, including InstallRequiredSystemLibraries in the
  # top-level CMakeLists.txt is sufficient. For mingw, we have to copy
  # mingw libraries in ourselves.
  set(ONE_GNU_DLL extra-dlls/libstdc++-6.dll)
  add_custom_command(OUTPUT ${ONE_GNU_DLL}
    COMMAND
    pwsh.exe -NoProfile -ExecutionPolicy Bypass -Command
    "${qpdf_SOURCE_DIR}/copy_dlls.ps1"
    -exe $<TARGET_FILE:qpdf>
    -dest "${CMAKE_CURRENT_BINARY_DIR}/extra-dlls"
    -mingwBinDir "${MINGW_BIN_DIR}"
    VERBATIM
    COMMAND_EXPAND_LISTS
    DEPENDS qpdf)
  add_custom_target(extra_dlls ALL DEPENDS ${ONE_GNU_DLL})
  add_dependencies(extra_dlls qpdf)
  if(BUILD_SHARED_LIBS)
    set(LIBQPDF_NAME get_filename_component($<TARGET_FILE:libqpdf> NAME))
    add_custom_command(
        TARGET qpdf POST_BUILD
        COMMAND
            ${CMAKE_COMMAND} -E copy_if_different
                $<TARGET_FILE:libqpdf>
                $<TARGET_FILE_DIR:qpdf>
        VERBATIM
        COMMAND_EXPAND_LISTS)
    set(EXTRA_DLL_COMPONENT ${COMPONENT_LIB})
  else()
    set(EXTRA_DLL_COMPONENT ${COMPONENT_CLI})
  endif()
  # The trailing / prevents "extra-dlls" from being created in bin.
  install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/extra-dlls/
    TYPE BIN
    COMPONENT ${EXTRA_DLL_COMPONENT})
endif()
