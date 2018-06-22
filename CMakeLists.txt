project(FreeImage C CXX)
cmake_minimum_required(VERSION 3.11)

# Extracting info from FreeImagePlus make based build
file(READ "${CMAKE_CURRENT_LIST_DIR}/fipMakefile.srcs" FI_MAKEFILE)

# Common makefile extraction code
string(REGEX REPLACE ".*VER_MAJOR = ([^\n]*).*" "\\1" FREEIMAGE_VERSION_MAJOR "${FI_MAKEFILE}")
string(REGEX REPLACE ".*VER_MINOR = ([^\n]*).*" "\\1" FREEIMAGE_VERSION_MINOR "${FI_MAKEFILE}")

message("VERSION : ${FREEIMAGE_VERSION_MAJOR}.${FREEIMAGE_VERSION_MINOR}")

set(VERSION ${FREEIMAGE_VERSION_MAJOR}.${FREEIMAGE_VERSION_MINOR})

string(REGEX REPLACE ".*SRCS = ([^\n]*).*" "\\1" FI_SOURCES "${FI_MAKEFILE}")
message("FI_SOURCES : ${FI_SOURCES}")

string(REGEX REPLACE ".*INCLUDE = -I([^\n]*).*" "\\1" FI_INCLUDES "${FI_MAKEFILE}")
message("FI_INCLUDES : ${FI_INCLUDES}")

# changing a string into a list
string(REPLACE " " ";" FI_SOURCES_LIST "${FI_SOURCES}")
string(REPLACE " -I" ";" FI_INCLUDES_LIST "${FI_INCLUDES}")

# Options to customize freeimage build
# TODO : enalbe/disable dependencies
# TODO :allow overriding embedded dependencies with "found" libraries

option(ENABLE_TESTS "Also build tests" 1)

#option(FREEIMAGE_DYNAMIC_C_RUNTIME "If ON build FreeImage with dynamicly linked C/C++ runtime. If OFF FreeImage is staticly linked with C/C++ runtime." 1)
#
#IF(NOT FREEIMAGE_DYNAMIC_C_RUNTIME)
#    # Set to link staticly with runtime
#    set(CMAKE_USER_MAKE_RULES_OVERRIDE ${CMAKE_CURRENT_SOURCE_DIR}/c_flag_overrides.cmake)
#    set(CMAKE_USER_MAKE_RULES_OVERRIDE_CXX ${CMAKE_CURRENT_SOURCE_DIR}/cxx_flag_overrides.cmake)
#ENDIF(NOT FREEIMAGE_DYNAMIC_C_RUNTIME)

# Check parameters
INCLUDE(CheckIncludeFile)
INCLUDE(CheckIncludeFileCXX)
INCLUDE(CheckFunctionExists)
INCLUDE(CheckTypeSize)

# Set default output dirs
#set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
#set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
#set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

#SET(FREEIMAGE_INCLUDE_DIRS ${CMAKE_SOURCE_DIR}/Source)
#SET(FREEIMAGE_LIBRARY_DIRS ${CMAKE_BINARY_DIR}/Lib)


add_library(FreeImageObjects OBJECT ${FI_SOURCES_LIST})
target_include_directories(FreeImageObjects PRIVATE ${FI_INCLUDES_LIST})
target_compile_definitions(FreeImageObjects PRIVATE -D__ANSI__)
set_target_properties(FreeImageObjects PROPERTIES
  POSITION_INDEPENDENT_CODE ON
  CXX_STANDARD 11
)

# platform specific options
# The goal is to match the original FreeImage Makefile structure
# but in a CMake way.
if(UNIX)
  set(libname "freeimage")
  # compiler specific options
  if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    target_compile_definitions(FreeImageObjects PRIVATE)
    target_compile_options(FreeImageObjects PRIVATE
      -fexceptions
      -fvisibility=hidden
      $<$<COMPILE_LANGUAGE:CXX>:-Wno-ctor-dtor-privacy>
    )
  elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    target_compile_definitions(FreeImageObjects PRIVATE OPJ_STATIC NO_LCMS DISABLE_PERF_MEASUREMENT)
    target_compile_options(FreeImageObjects PRIVATE
      -fexceptions
      -fvisibility=hidden
      $<$<COMPILE_LANGUAGE:CXX>:-Wno-ctor-dtor-privacy>
    )
  elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    target_compile_definitions(FreeImageObjects PRIVATE _CRT_SECURE_NO_WARNINGS LIBRAW_NODLL)
  endif()
elseif(WIN32)
elseif(APPLE)
  set(libname "freeimage-${VERSION}")
else()
endif()


# preparing layout to match de-facto C++ standard (for easy include as dependency, even for this build)
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Source/FreeImage.h ${CMAKE_CURRENT_SOURCE_DIR}/include/FreeImage.h COPYONLY)

# Defining final targets
add_library(FreeImageShared SHARED
  $<TARGET_OBJECTS:FreeImageObjects>
)
set_target_properties(FreeImageShared PROPERTIES OUTPUT_NAME ${libname})
target_include_directories(FreeImageShared PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>  # <prefix>/include
)
install(TARGETS FreeImageShared DESTINATION lib)

add_library(FreeImageStatic STATIC
  $<TARGET_OBJECTS:FreeImageObjects>
)
set_target_properties(FreeImageStatic PROPERTIES OUTPUT_NAME ${libname})
target_include_directories(FreeImageStatic PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>  # <prefix>/include
)
install(TARGETS FreeImageStatic DESTINATION lib)

# target specific options
target_compile_definitions(FreeImageShared PUBLIC -DFREEIMAGE_LIB)

#target_compile_definitions(FreeImageObjects PRIVATE
#  NO_LCMS
#  OPJ_STATIC
#  DISABLE_PERF_MEASUREMENT
#  __ANSI__
#  PNG_ARM_NEON_OPT=0
#  $<$<PLATFORM_ID:Windows>:_CRT_SECURE_NO_WARNINGS FREEIMAGE_LIB LIBRAW_NODLL>
#)
#target_compile_options(FreeImageObjects PRIVATE
#  $<$<NOT:$<PLATFORM_ID:Windows>>:-fexceptions $<$<COMPILE_LANGUAGE:CXX>:-Wno-ctor-dtor-privacy>>
#  $<$<PLATFORM_ID:Windows>:/wd4101 /wd4477 /wd4244 /wd4018 /wd4305> # clean code on windows :(
#)


if(ENABLE_TESTS)
  ENABLE_TESTING()

  set(TEST_SOURCES
    TestAPI/MainTestSuite.cpp
    TestAPI/testHeaderOnly.cpp
    TestAPI/testChannels.cpp
    TestAPI/testImageType.cpp
    TestAPI/testMemIO.cpp
    TestAPI/testMPage.cpp
    TestAPI/testMPageMemory.cpp
    TestAPI/testMPageStream.cpp
    TestAPI/testPlugins.cpp
    TestAPI/testThumbnail.cpp
    TestAPI/testTools.cpp
    TestAPI/testWrappedBuffer.cpp
    TestAPI/testJPEG.cpp
    TestAPI/TestSuite.h
  )

  add_executable(FreeImage_TestAPI ${TEST_SOURCES})
  target_link_libraries(FreeImage_TestAPI FreeImageStatic)
  add_test(NAME Test COMMAND FreeImage_TestAPI WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/TestAPI)
endif()

