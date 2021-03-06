# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

add_custom_target(plasma-all)
add_custom_target(plasma)
add_custom_target(plasma-benchmarks)
add_custom_target(plasma-tests)
add_dependencies(plasma-all plasma plasma-tests plasma-benchmarks)

# For the moment, Plasma is versioned like Arrow
project(plasma VERSION "${ARROW_BASE_VERSION}")
set(PLASMA_VERSION "${ARROW_VERSION}")

find_package(Threads)

# The SO version is also the ABI version
set(PLASMA_SO_VERSION "${ARROW_SO_VERSION}")
set(PLASMA_FULL_SO_VERSION "${ARROW_FULL_SO_VERSION}")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-conversion")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")

# Set compiling options for asio headers.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-conversion -Wno-documentation")

set(PLASMA_IO_SRCS
    fling.cc
    io/basic_connection.cc
    io/connection.cc
    protocol.cc)

set(PLASMA_SRCS
    client.cc
    common.cc
    malloc.cc
    plasma.cc
    protocol.cc
    ${PLASMA_IO_SRCS})

set(PLASMA_STORE_SRCS
    dlmalloc.cc
    eviction_policy.cc
    quota_aware_policy.cc
    plasma_allocator.cc
    ${PLASMA_IO_SRCS}
    store.cc)

set(PLASMA_LINK_LIBS arrow_shared)
set(PLASMA_STATIC_LINK_LIBS arrow_static)

if(ARROW_CUDA)
  set(PLASMA_LINK_LIBS ${PLASMA_LINK_LIBS} arrow_cuda_shared)
  set(PLASMA_STATIC_LINK_LIBS arrow_cuda_static ${PLASMA_STATIC_LINK_LIBS})
  add_definitions(-DPLASMA_CUDA)
endif()

if(NOT APPLE AND NOT MSVC)
  # Localize thirdparty symbols using a linker version script. This hides them
  # from the client application. The OS X linker does not support the
  # version-script option.
  set(PLASMA_SHARED_LINK_FLAGS
      "-Wl,--no-undefined,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/symbols.map")
endif()

add_library(plasma_shared SHARED ${PLASMA_SRCS})
set_target_properties(plasma_shared 
                      PROPERTIES  LIBRARY_OUTPUT_DIRECTORY
                                  "${BUILD_OUTPUT_ROOT_DIRECTORY}"
                                  RUNTIME_OUTPUT_DIRECTORY
                                  "${BUILD_OUTPUT_ROOT_DIRECTORY}"
                                  PDB_OUTPUT_DIRECTORY
                                  "${BUILD_OUTPUT_ROOT_DIRECTORY}"
                                  OUTPUT_NAME
                                  "plasma"
                                  VERSION
                                  "${ARROW_FULL_SO_VERSION}"
                                  SOVERSION
                                  "${ARROW_SO_VERSION}")
target_link_libraries(plasma_shared ${BOOST_SYSTEM_LIBRARY} ${PLASMA_LINK_LIBS} )
install(TARGETS plasma_shared ${INSTALL_IS_OPTIONAL}
        EXPORT plasma_targets
        RUNTIME DESTINATION ${RUNTIME_INSTALL_DIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

add_library(plasma_static STATIC ${PLASMA_SRCS})
set_target_properties(plasma_static
                      PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}" 
                      OUTPUT_NAME "plasma")
target_link_libraries(plasma_static ${BOOST_SYSTEM_LIBRARY} ${PLASMA_STATIC_LINK_LIBS} )
install(TARGETS plasma_static ${INSTALL_IS_OPTIONAL}
        EXPORT plasma_targets
        RUNTIME DESTINATION ${RUNTIME_INSTALL_DIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})


foreach(LIB_TARGET ${PLASMA_LIBRARIES})
  target_compile_definitions(${LIB_TARGET} PRIVATE ARROW_EXPORTING)
endforeach()

# The optimization flag -O3 is suggested by dlmalloc.c, which is #included in
# malloc.cc; we set it here regardless of whether we do a debug or release build.
set_source_files_properties(dlmalloc.cc PROPERTIES COMPILE_FLAGS "-O3")

if(CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang" OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  set_property(SOURCE dlmalloc.cc
               APPEND_STRING
               PROPERTY COMPILE_FLAGS " -Wno-parentheses-equality \
-Wno-null-pointer-arithmetic \
-Wno-shorten-64-to-32 \
-Wno-unused-macros")
endif()

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set_property(SOURCE dlmalloc.cc APPEND_STRING PROPERTY COMPILE_FLAGS " -Wno-conversion")
endif()

list(APPEND PLASMA_EXTERNAL_STORE_SOURCES "external_store.cc" "hash_table_store.cc")

# We use static libraries for the plasma-store-server executable so that it can
# be copied around and used in different locations.
add_executable(plasma-store-server ${PLASMA_EXTERNAL_STORE_SOURCES} ${PLASMA_STORE_SRCS})
if(ARROW_BUILD_STATIC)
  target_link_libraries(plasma-store-server plasma_static ${PLASMA_STATIC_LINK_LIBS})
else()
  # Fallback to shared libs in the case that static libraries are not build.
  target_link_libraries(plasma-store-server plasma_shared ${PLASMA_LINK_LIBS})
endif()

add_dependencies(plasma plasma-store-server)
if(ARROW_RPATH_ORIGIN)
  if(APPLE)
    set(_lib_install_rpath "@loader_path")
  else()
    set(_lib_install_rpath "\$ORIGIN")
  endif()
  set_target_properties(plasma-store-server
                        PROPERTIES INSTALL_RPATH ${_lib_install_rpath})
elseif(APPLE)
  # With OSX and conda, we need to set the correct RPATH so that dependencies
  # are found. The installed libraries with conda have an RPATH that matches
  # for executables and libraries lying in $ENV{CONDA_PREFIX}/bin or
  # $ENV{CONDA_PREFIX}/lib but our test libraries and executables are not
  # installed there.
  if(NOT "$ENV{CONDA_PREFIX}" STREQUAL "" AND APPLE)
    set_target_properties(plasma-store-server
                          PROPERTIES BUILD_WITH_INSTALL_RPATH
                                     TRUE
                                     INSTALL_RPATH_USE_LINK_PATH
                                     TRUE
                                     INSTALL_RPATH
                                     "$ENV{CONDA_PREFIX}/lib")
  endif()
endif()

install(FILES common.h
              compat.h
              client.h
              test_util.h
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/plasma")

# Plasma store
set_target_properties(plasma-store-server PROPERTIES INSTALL_RPATH_USE_LINK_PATH TRUE)
install(TARGETS plasma-store-server ${INSTALL_IS_OPTIONAL} DESTINATION
                ${CMAKE_INSTALL_BINDIR})

if(ARROW_PLASMA_JAVA_CLIENT)
  # Plasma java client support
  find_package(JNI REQUIRED)
  # add jni support
  include_directories(${JAVA_INCLUDE_PATH})
  include_directories(${JAVA_INCLUDE_PATH2})
  if(JNI_FOUND)
    message(STATUS "JNI_INCLUDE_DIRS = ${JNI_INCLUDE_DIRS}")
    message(STATUS "JNI_LIBRARIES = ${JNI_LIBRARIES}")
  else()
    message(WARNING "Could not find JNI")
  endif()

  add_compile_options("-I$ENV{JAVA_HOME}/include/")
  if(WIN32)
    add_compile_options("-I$ENV{JAVA_HOME}/include/win32")
  elseif(APPLE)
    add_compile_options("-I$ENV{JAVA_HOME}/include/darwin")
  else() # linux
    add_compile_options("-I$ENV{JAVA_HOME}/include/linux")
  endif()

  include_directories("${CMAKE_CURRENT_LIST_DIR}/lib/java")

  file(GLOB PLASMA_LIBRARY_EXT_java_SRC lib/java/*.cc lib/*.cc)
  add_library(plasma_java SHARED ${PLASMA_LIBRARY_EXT_java_SRC})

  if(APPLE)
    target_link_libraries(plasma_java
                          plasma_shared
                          ${PLASMA_LINK_LIBS}
                          "-undefined dynamic_lookup"
                          ${PTHREAD_LIBRARY}
                          )
  else(APPLE)
    target_link_libraries(plasma_java
                          plasma_shared
                          ${PLASMA_LINK_LIBS}
                          ${PTHREAD_LIBRARY}
                          )
  endif(APPLE)

  install(TARGETS plasma_java ${INSTALL_IS_OPTIONAL}
        RUNTIME DESTINATION ${RUNTIME_INSTALL_DIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endif()
#
# Unit tests
#

# Adding unit tests part of the "arrow" portion of the test suite
function(ADD_PLASMA_TEST REL_TEST_NAME)
  set(options)
  set(one_value_args)
  set(multi_value_args)
  cmake_parse_arguments(ARG
                        "${options}"
                        "${one_value_args}"
                        "${multi_value_args}"
                        ${ARGN})
  add_test_case(${REL_TEST_NAME}
                PREFIX
                "plasma"
                LABELS
                "plasma-tests"
                ${ARG_UNPARSED_ARGUMENTS})
endfunction()

if(ARROW_BUILD_SHARED)
  set(PLASMA_TEST_LIBS plasma_shared ${PLASMA_LINK_LIBS})
else()
  set(PLASMA_TEST_LIBS plasma_static ${PLASMA_STATIC_LINK_LIBS})
endif()
     
add_plasma_test(test/serialization_tests EXTRA_LINK_LIBS ${PLASMA_TEST_LIBS})
add_plasma_test(test/client_tests
                EXTRA_LINK_LIBS
                ${PLASMA_TEST_LIBS}
                EXTRA_DEPENDENCIES
                plasma-store-server)
