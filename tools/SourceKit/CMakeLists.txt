include(CheckSymbolExists)

# Append our own modules to the module path.
list(APPEND CMAKE_MODULE_PATH
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
  message(SEND_ERROR "SourceKit cannot be built standalone")
endif()

if(NOT CMAKE_C_COMPILER_ID MATCHES Clang)
  if(CMAKE_SYSTEM_NAME STREQUAL CMAKE_HOST_SYSTEM_NAME)
    get_target_property(CLANG_LOCATION clang LOCATION)
    get_filename_component(CLANG_LOCATION ${CLANG_LOCATION} DIRECTORY)

    if(CMAKE_C_COMPILER_ID STREQUAL MSVC OR CMAKE_C_SIMULATE_ID STREQUAL MSVC)
      set(CMAKE_C_COMPILER_ID
        ${CLANG_LOCATION}/clang-cl${CMAKE_EXECUTABLE_SUFFIX})
      set(CMAKE_CXX_COMPILER_ID
        ${CLANG_LOCATION}/clang-cl${CMAKE_EXECUTABLE_SUFFIX})
    else()
      set(CMAKE_C_COMPILER_ID
        ${CLANG_LOCATION}/clang${CMAKE_EXECUTABLE_SUFFIX})
      set(CMAKE_CXX_COMPILER_ID
        ${CLANG_LOCATION}/clang++${CMAKE_EXECUTABLE_SUFFIX})
    endif()
  else()
    message(SEND_ERROR "SourceKit requires a clang based compiler")
  endif()
endif()

if (DARWIN_TOOLCHAIN_VERSION)
  set(SOURCEKIT_VERSION_STRING "${DARWIN_TOOLCHAIN_VERSION}")
else()
  set(SOURCEKIT_VERSION_STRING "${SWIFT_COMPILER_VERSION}")
endif()

set(SOURCEKIT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
set(SOURCEKIT_BINARY_DIR ${SWIFT_BINARY_DIR})

set(SOURCEKIT_RUNTIME_OUTPUT_INTDIR "${SWIFT_RUNTIME_OUTPUT_INTDIR}")
set(SOURCEKIT_LIBRARY_OUTPUT_INTDIR "${SWIFT_LIBRARY_OUTPUT_INTDIR}")

check_symbol_exists(dispatch_block_create "dispatch/dispatch.h" HAVE_DISPATCH_BLOCK_CREATE)
configure_file(
  ${SOURCEKIT_SOURCE_DIR}/include/SourceKit/Config/config.h.cmake
  ${SOURCEKIT_BINARY_DIR}/include/SourceKit/Config/config.h)

set(SOURCEKIT_DEPLOYMENT_OS "${SWIFT_HOST_VARIANT}")
set(SOURCEKIT_DEPLOYMENT_TARGET "${SWIFT_SDK_${SWIFT_HOST_VARIANT_SDK}_DEPLOYMENT_VERSION}")

STRING(TOLOWER "${SWIFT_PRIMARY_VARIANT_SDK}" SOURCEKIT_PLATFORM_NAME)
# Remove '_simulator' from the name if it exists
STRING(REPLACE "_simulator" "" SOURCEKIT_PLATFORM_NAME "${SOURCEKIT_PLATFORM_NAME}")

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin" AND NOT CMAKE_CROSSCOMPILING)
  set(CMAKE_OSX_SYSROOT "${SWIFT_SDK_${SWIFT_HOST_VARIANT_SDK}_PATH}")
  set(CMAKE_OSX_ARCHITECTURES "${SWIFT_HOST_VARIANT_ARCH}")
  set(CMAKE_OSX_DEPLOYMENT_TARGET "")
endif()

# If we were don't have XPC, just build inproc.
if(NOT "${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin" OR NOT HAVE_XPC_H)
  set(SWIFT_SOURCEKIT_USE_INPROC_LIBRARY TRUE)
endif()

# Now include AddSwiftSourceKit
include(AddSwiftSourceKit)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  # Choose a deployment target if none was set.
  set(SOURCEKIT_DEPLOYMENT_TARGET "" CACHE STRING
      "Deployment target for SourceKit.")
  if (NOT SOURCEKIT_DEPLOYMENT_TARGET)
    execute_process(COMMAND sw_vers -productVersion
                    OUTPUT_VARIABLE SOURCEKIT_DEPLOYMENT_TARGET
                    OUTPUT_STRIP_TRAILING_WHITESPACE)
    string(REGEX MATCH "[0-9]+\\.[0-9]+" SOURCEKIT_DEPLOYMENT_TARGET "${SOURCEKIT_DEPLOYMENT_TARGET}")
  endif()

  # Sadly there are two OS naming conventions.
  # xcrun SDK name:   macosx iphoneos iphonesimulator (+ "internal" or version)
  # -mOS-version-min: macosx ios      ios-simulator

  if (SOURCEKIT_DEPLOYMENT_OS MATCHES "^iphoneos")
    set(version_min_os "ios")
    set(triple_os "ios")
    set(SOURCEKIT_DEFAULT_TARGET_SDK "IOS")
  elseif (SOURCEKIT_DEPLOYMENT_OS MATCHES "^iphonesimulator")
    set(version_min_os "ios-simulator")
    set(triple_os "ios")
    set(SOURCEKIT_DEFAULT_TARGET_SDK "IOS_SIMULATOR")
  elseif (SOURCEKIT_DEPLOYMENT_OS MATCHES "^macosx")
    set(version_min_os "macosx")
    set(triple_os "macosx")
    set(SOURCEKIT_DEFAULT_TARGET_SDK "OSX")
  else()
    message(FATAL_ERROR "Can't understand SOURCEKIT_DEPLOYMENT_OS '${SOURCEKIT_DEPLOYMENT_OS}';")
  endif()

  # Add deployment target to C/C++ compiler and linker flags.
  # FIXME: CMAKE_OSX_DEPLOYMENT_TARGET falls over when used for iOS versions.
  if (XCODE)
    if (${SOURCEKIT_DEPLOYMENT_OS} MATCHES "^macosx")
      set(CMAKE_XCODE_ATTRIBUTE_MACOSX_DEPLOYMENT_TARGET ${SOURCEKIT_DEPLOYMENT_TARGET})
    else()
      set(CMAKE_XCODE_ATTRIBUTE_IPHONEOS_DEPLOYMENT_TARGET ${SOURCEKIT_DEPLOYMENT_TARGET})
    endif()
  else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -m${version_min_os}-version-min=${SOURCEKIT_DEPLOYMENT_TARGET}")
  endif()
endif()

include_directories(BEFORE
  ${SOURCEKIT_BINARY_DIR}/include
  ${SOURCEKIT_SOURCE_DIR}/include
  ${SOURCEKIT_BINARY_DIR}/tools/SourceKit/include
)

if(NOT "${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
  string(TOUPPER "${CMAKE_SYSTEM_NAME}" SOURCEKIT_DEFAULT_TARGET_SDK)
endif()

add_subdirectory(include)
add_subdirectory(lib)
add_subdirectory(tools)
