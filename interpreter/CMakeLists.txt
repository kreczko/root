
#---Define the way we want to build and what of llvm/clang/cling------------------------------------

set(LLVM_INCLUDE_TESTS OFF CACHE BOOL "")
set(LLVM_INCLUDE_EXAMPLES OFF)
set(LLVM_BUILD_TOOLS OFF)
set(LLVM_FORCE_USE_OLD_TOOLCHAIN ON)
# We only use llvm/clang through TCling which is (with the help of core/meta) already taking a lock
# to serialize access to llvm.  We can later review how to make this finer grained by using llvm's own locking
# mechanism.
set(LLVM_ENABLE_THREADS OFF)

if(ROOT_ARCHITECTURE MATCHES linuxarm64)
  set(ROOT_CLING_TARGET "AArch64")
elseif(ROOT_ARCHITECTURE MATCHES linux)
  set(ROOT_CLING_TARGET "X86")
elseif(ROOT_ARCHITECTURE MATCHES macosx)
  set(ROOT_CLING_TARGET "X86")
else()
  set(ROOT_CLING_TARGET "all")
endif()

set(LLVM_TARGETS_TO_BUILD ${ROOT_CLING_TARGET} CACHE STRING "Semicolon-separated list of targets to build, or \"all\".")

#---Protect against warnings clang would fire-------------------------------------------------------

if(${CMAKE_CXX_COMPILER_ID} STREQUAL Clang)
   set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-conditional-uninitialized")
endif()

#---Do not transform warnings in errors-------------------------------------------------------------
string(REPLACE "-Werror" "" CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})

#---Build type--------------------------------------------------------------------------------------
if(NOT DEFINED LLVM_BUILD_TYPE)
  set(LLVM_BUILD_TYPE Release CACHE STRING "Build type for LLVM (used to set CMAKE_BUILD_TYPE)")
endif()
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(LLVM_ENABLE_ASSERTIONS "YES")
else()
  if(LLVM_BUILD_TYPE STREQUAL "Debug")
    set(LLVM_ENABLE_ASSERTIONS "YES")
  else()
    set(LLVM_ENABLE_ASSERTIONS "NO")
  endif()
endif()
set(CMAKE_BUILD_TYPE ${LLVM_BUILD_TYPE})

#---Add the sub-directory excluding all the targets from all-----------------------------------------
if(CMAKE_GENERATOR MATCHES "Xcode")
  add_subdirectory(llvm/src)
else()
  add_subdirectory(llvm/src EXCLUDE_FROM_ALL)
endif()

#---Mark the LLVM/CLANG variables as advanced--------------------------------------------------------
get_cmake_property(variables CACHE_VARIABLES)
foreach(var ${variables})
  if(var MATCHES "^(CLANG|LLVM|CLING)_")
    mark_as_advanced(FORCE ${var})
  endif()
endforeach()
mark_as_advanced(FORCE BUG_REPORT_URL BUILD_CLANG_FORMAT_VS_PLUGIN BUILD_SHARED_LIBS BUILD_TESTING
                       C_INCLUDE_DIRS DEFAULT_SYSROOT FFI_INCLUDE_DIR FFI_LIBRARY_DIR
                       GCC_INSTALL_PREFIX LIBCLANG_BUILD_STATIC TOOL_INFO_PLIST)
mark_as_advanced(CLEAR LLVM_ENABLE_ASSERTIONS LLVM_BUILD_TYPE)
#---Set into parent scope LLVM_VERSION --------------------------------------------------------------
file(READ ${CMAKE_CURRENT_SOURCE_DIR}/llvm/src/configure _filestr)
string(REGEX REPLACE ".*PACKAGE_VERSION='([0-9]+[.][0-9]+[.][0-9]+).*" "\\1" _llvm_version ${_filestr})
set(LLVM_VERSION ${_llvm_version} PARENT_SCOPE)

#---Added some missing dependencies------------------------------------------------------------------
add_dependencies(clingUtils clangCodeGen)
add_dependencies(clingInterpreter clangBasic clangAST ClangDriverOptions)
add_dependencies(clingMetaProcessor clangAST ClangDriverOptions)
