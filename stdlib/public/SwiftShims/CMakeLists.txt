set(sources
  AssertionReporting.h
  CoreFoundationShims.h
  FoundationShims.h
  GlobalObjects.h
  HeapObject.h
  KeyPath.h
  LibcOverlayShims.h
  LibcShims.h
  Random.h
  RefCount.h
  RuntimeShims.h
  RuntimeStubs.h
  SwiftStdbool.h
  SwiftStddef.h
  SwiftStdint.h
  System.h
  Target.h
  ThreadLocalStorage.h
  UnicodeShims.h
  Visibility.h

  DispatchOverlayShims.h
  NetworkOverlayShims.h
  OSOverlayShims.h
  ObjectiveCOverlayShims.h
  SafariServicesOverlayShims.h
  UIKitOverlayShims.h
  XCTestOverlayShims.h
  XPCOverlayShims.h

  CoreFoundationOverlayShims.h
  CFCharacterSetShims.h
  CFHashingShims.h

  FoundationOverlayShims.h
  FoundationShimSupport.h
  NSCalendarShims.h
  NSCharacterSetShims.h
  NSCoderShims.h
  NSDataShims.h
  NSDictionaryShims.h
  NSErrorShims.h
  NSFileManagerShims.h
  NSIndexPathShims.h
  NSIndexSetShims.h
  NSKeyedArchiverShims.h
  NSLocaleShims.h
  NSTimeZoneShims.h
  NSUndoManagerShims.h

  ClockKitOverlayShims.h

  module.modulemap
  )
set(output_dir "${SWIFTLIB_DIR}/shims")

add_custom_command(
    OUTPUT "${output_dir}"
    COMMAND ${CMAKE_COMMAND} "-E" "make_directory" "${output_dir}")
set(outputs)
foreach(input ${sources})
  add_custom_command(
      OUTPUT "${output_dir}/${input}"
      DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/${input}"
      COMMAND
        "${CMAKE_COMMAND}" "-E" "copy_if_different"
        "${CMAKE_CURRENT_SOURCE_DIR}/${input}"
        "${output_dir}/${input}"
      COMMENT "Copying ${input} to ${output_dir}")
  list(APPEND outputs "${output_dir}/${input}")
endforeach()
# Put the output dir itself last so that it isn't considered the primary output.
list(APPEND outputs "${output_dir}")

add_custom_target("copy_shim_headers"
    DEPENDS "${outputs}"
    COMMENT "Copying SwiftShims module to ${output_dir}")

if ("${LLVM_PACKAGE_VERSION}" STREQUAL "")
  message(FATAL_ERROR
          "LLVM_PACKAGE_VERSION must be set before including subdirectories")
endif()

# Symlink in the Clang headers.
# First extract the "version" used for Clang's resource directory.
string(REGEX MATCH "[0-9]+\\.[0-9]+(\\.[0-9]+)?" CLANG_VERSION
  "${LLVM_PACKAGE_VERSION}")
if(NOT SWIFT_INCLUDE_TOOLS AND SWIFT_BUILD_RUNTIME_WITH_HOST_COMPILER)
  if(SWIFT_COMPILER_IS_MSVC_LIKE)
    execute_process(COMMAND ${CMAKE_C_COMPILER} /clang:-print-resource-dir
      OUTPUT_VARIABLE clang_headers_location
      OUTPUT_STRIP_TRAILING_WHITESPACE
      ERROR_QUIET)
  else()
    execute_process(COMMAND ${CMAKE_C_COMPILER} -print-resource-dir
      OUTPUT_VARIABLE clang_headers_location
      OUTPUT_STRIP_TRAILING_WHITESPACE
      ERROR_QUIET)
  endif()
  message(STATUS "Using clang Resource Directory: ${clang_headers_location}")
else()
  set(clang_headers_location "${LLVM_LIBRARY_OUTPUT_INTDIR}/clang/${CLANG_VERSION}")
endif()

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Windows")
  set(cmake_symlink_option "copy_directory")
else()
  set(cmake_symlink_option "create_symlink")
endif()

add_custom_command_target(unused_var
    COMMAND
      "${CMAKE_COMMAND}" "-E" "make_directory" "${SWIFTLIB_DIR}"
    COMMAND
      "${CMAKE_COMMAND}" "-E" "${cmake_symlink_option}"
      "${clang_headers_location}"
      "${SWIFTLIB_DIR}/clang"

    CUSTOM_TARGET_NAME "symlink_clang_headers"
    OUTPUT "${SWIFTLIB_DIR}/clang"
    COMMENT "Symlinking Clang resource headers into ${SWIFTLIB_DIR}/clang")
add_dependencies(copy_shim_headers symlink_clang_headers)

swift_install_in_component(FILES ${sources}
                           DESTINATION "lib/swift/shims"
                           COMPONENT stdlib)

# Install Clang headers under the Swift library so that an installed Swift's
# module importer can find the compiler headers corresponding to its Clang.
swift_install_in_component(DIRECTORY "${clang_headers_location}/"
                           DESTINATION "lib/swift/clang"
                           COMPONENT clang-builtin-headers
                           PATTERN "*.h")

swift_install_symlink_component(clang-resource-dir-symlink
  LINK_NAME clang
  TARGET ../clang/${CLANG_VERSION}
  DESTINATION "lib/swift")

# Possibly install Clang headers under Clang's resource directory in case we
# need to use a different version of the headers than the installed Clang. This
# should be used in conjunction with clang-resource-dir-symlink.
file(TO_CMAKE_PATH "${LLVM_LIBRARY_OUTPUT_INTDIR}"
  _SWIFT_SHIMS_PATH_TO_CLANG_BUILD)
swift_install_in_component(DIRECTORY "${_SWIFT_SHIMS_PATH_TO_CLANG_BUILD}/lib/clang"
                           DESTINATION "lib"
                           COMPONENT clang-builtin-headers-in-clang-resource-dir
                           PATTERN "*.h")
