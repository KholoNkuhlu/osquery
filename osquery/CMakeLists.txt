# osquery core source files included with the SDK (libosquery).
set(OSQUERY_SOURCES "")
set(OSQUERY_LINKS "")
set(OSQUERY_TESTS "")
set(OSQUERY_KERNEL_TESTS "")

# osquery benchmarking sources
set(OSQUERY_BENCHMARKS "")
set(OSQUERY_KERNEL_BENCHMARKS "")

# osquery core additional sources files not included with SDK (libosquery_additional).
set(OSQUERY_ADDITIONAL_SOURCES "")
set(OSQUERY_ADDITIONAL_LINKS "")
set(OSQUERY_ADDITIONAL_TESTS "")
set(OSQUERY_TABLES_TESTS "")

# Add all and extra for osquery code.
# TODO(WIN64): Changes for Win64 CMake
if(WIN32)
  add_compile_options(
    /W3
  )
else()
  add_compile_options(
    -Wall
    -Wextra
    -Wno-unused-parameter
  )
endif()

if(WIN32)
  ADD_OSQUERY_LINK_CORE("netapi32.lib")
  ADD_OSQUERY_LINK_CORE("rpcrt4.lib")
  ADD_OSQUERY_LINK_CORE("shlwapi.lib")
  ADD_OSQUERY_LINK_CORE("wbemuuid.lib")
else()
  ADD_OSQUERY_LINK_CORE("libpthread")
  ADD_OSQUERY_LINK_CORE("libz")
endif()

if(APPLE OR LINUX)
  ADD_OSQUERY_LINK_CORE("libdl")
elseif(FREEBSD)
  ADD_OSQUERY_LINK_CORE("icuuc")
endif()

if(LINUX OR FREEBSD)
  ADD_OSQUERY_LINK_CORE("librt")
  ADD_OSQUERY_LINK_CORE("libc")
endif()

ADD_OSQUERY_LINK_CORE("rocksdb_lite")
ADD_OSQUERY_LINK_CORE("snappy")
ADD_OSQUERY_LINK_CORE("lz4")
ADD_OSQUERY_LINK_CORE("thrift")
ADD_OSQUERY_LINK_CORE("gflags")
ADD_OSQUERY_LINK_CORE("glog")
ADD_OSQUERY_LINK_CORE("boost_system-mt")
ADD_OSQUERY_LINK_CORE("boost_filesystem-mt")
ADD_OSQUERY_LINK_CORE("boost_regex-mt")

# TODO(#1956): Ignoring on WIN32 for now
if(NOT WIN32)
  ADD_OSQUERY_LINK_ADDITIONAL("yara")
endif()

# Remaining additional development libraries.
ADD_OSQUERY_LINK_ADDITIONAL("cppnetlib-uri")
ADD_OSQUERY_LINK_ADDITIONAL("cppnetlib-client-connections")
ADD_OSQUERY_LINK_ADDITIONAL("ssl")
ADD_OSQUERY_LINK_ADDITIONAL("crypto")
ADD_OSQUERY_LINK_ADDITIONAL("readline")
ADD_OSQUERY_LINK_ADDITIONAL("ncurses")

if(APPLE)
  ADD_OSQUERY_LINK_CORE("liblzma libbz2")
else()
  ADD_OSQUERY_LINK_CORE("lzma bz2")
  ADD_OSQUERY_LINK_ADDITIONAL("bz2")
endif()

# The platform-specific SDK + core linker flags.
ADD_OSQUERY_LINK_CORE("-rdynamic")
if(APPLE)
  ADD_OSQUERY_LINK_CORE("-Wl,-dead_strip")
  ADD_OSQUERY_LINK_CORE("-mmacosx-version-min=${OSX_VERSION_MIN}")
elseif(LINUX OR FREEBSD)
  ADD_OSQUERY_LINK_CORE("-Wl,-zrelro -Wl,-znow")
  if(NOT ${REDHAT_BASED})
    ADD_OSQUERY_LINK_CORE("-Wl,--gc-sections")
  endif()
  if(NOT DEFINED ENV{SANITIZE})
    ADD_OSQUERY_LINK_CORE("-pie")
  endif()
endif()

if(LINUX)
  if(DEFINED ENV{OSQUERY_BUILD_SHARED})
    ADD_OSQUERY_LINK_CORE("stdc++")
    ADD_OSQUERY_LINK_ADDITIONAL("stdc++")
  endif()
  if(NOT DEFINED ENV{OSQUERY_BUILD_LINK_SHARED})
    ADD_OSQUERY_LINK_CORE("-static-libstdc++")
    ADD_OSQUERY_LINK_ADDITIONAL("-static-libstdc++")
  endif()
  # For Ubuntu/CentOS packages add the build SHA1.
  ADD_OSQUERY_LINK_CORE("-Wl,--build-id")
endif()

if(DEFINED ENV{SANITIZE})
  if(DEFINED ENV{SANITIZE_THREAD})
    ADD_OSQUERY_LINK_CORE(-fsanitize=thread)
  else()
    ADD_OSQUERY_LINK_CORE(-fsanitize=address -fsanitize=leak)
  endif()
  ADD_OSQUERY_LINK_CORE(-fsanitize-blacklist=${SANITIZE_BLACKLIST})
endif()

# Construct a set of all object files, starting with third-party and all
# of the osquery core objects (sources from ADD_CORE_LIBRARY macros).
set(OSQUERY_OBJECTS $<TARGET_OBJECTS:osquery_sqlite>)

# Add subdirectories
add_subdirectory(config)
add_subdirectory(core)
add_subdirectory(database)
add_subdirectory(devtools)
add_subdirectory(dispatcher)
add_subdirectory(distributed)
add_subdirectory(events)
add_subdirectory(extensions)
add_subdirectory(filesystem)
add_subdirectory(logger)
add_subdirectory(registry)
add_subdirectory(sql)

if(NOT WIN32)
  add_subdirectory(remote)
endif()

if(NOT DEFINED ENV{SKIP_TABLES})
  add_subdirectory(tables)

  # Amalgamate the utility tables needed to compile.
  GENERATE_UTILITIES("${CMAKE_SOURCE_DIR}")
  AMALGAMATE("${CMAKE_SOURCE_DIR}" "utils" AMALGAMATION_UTILS)
  ADD_OSQUERY_LIBRARY_CORE(osquery_amalgamation ${AMALGAMATION_UTILS})
endif()

# Bubble the subdirectory (component) sources and links for this build.
list(APPEND OSQUERY_OBJECTS ${OSQUERY_SOURCES})
list(APPEND OSQUERY_LIBS ${OSQUERY_LINKS})

set(CMAKE_MACOSX_RPATH 0)
set(CMAKE_SKIP_RPATH TRUE)

# Create the static libosquery (everything but non-utility tables).
add_library(libosquery STATIC main/lib.cpp ${OSQUERY_OBJECTS})
target_link_libraries(libosquery ${OSQUERY_LIBS})
set_target_properties(libosquery PROPERTIES OUTPUT_NAME osquery)
set_target_properties(libosquery PROPERTIES COMPILE_FLAGS
  "-DOSQUERY_BUILD_VERSION=${OSQUERY_BUILD_VERSION} ${CXX_COMPILE_FLAGS}")

# Create the dynamic libosquery.
if(DEFINED ENV{OSQUERY_BUILD_SHARED})
  add_library(libosquery_shared SHARED main/lib.cpp ${OSQUERY_OBJECTS})
  target_link_libraries(libosquery_shared ${OSQUERY_LIBS})
  SET_OSQUERY_COMPILE(libosquery_shared "${CXX_COMPILE_FLAGS}")
  set_target_properties(libosquery_shared PROPERTIES OUTPUT_NAME osquery)
endif()

# A friendly echo printed after the library is built.
add_custom_target(osquery_library ALL
  DEPENDS libosquery
  COMMAND ${CMAKE_COMMAND} -E cmake_echo_color --cyan --bold
    "-- Built libosquery: ${CMAKE_BINARY_DIR}/libosquery.a"
)

# make devel (implies install)
add_custom_target(devel
  COMMAND ${CMAKE_COMMAND}
    -D COMPONENT=devel
    -P cmake_install.cmake
    DEPENDS libosquery_basic
)
add_dependencies(devel libosquery)

if(NOT ${OSQUERY_BUILD_SDK_ONLY})
  if(NOT DEFINED ENV{SKIP_TABLES})
    # Generate the osquery additional tables (the non-util).
    GENERATE_TABLES("${CMAKE_SOURCE_DIR}")
    AMALGAMATE("${CMAKE_SOURCE_DIR}" "additional" AMALGAMATION)
    ADD_OSQUERY_LIBRARY_ADDITIONAL(osquery_additional_amalgamation ${AMALGAMATION})
  endif()

  AMALGAMATE("${CMAKE_SOURCE_DIR}" "foreign" AMALGAMATION_FOREIGN)
  ADD_OSQUERY_LIBRARY_ADDITIONAL(osquery_foreign_amalgamation ${AMALGAMATION_FOREIGN})

  # Create the static libosquery_additional.
  add_library(libosquery_additional STATIC ${OSQUERY_ADDITIONAL_SOURCES})
  target_link_libraries(libosquery_additional ${OSQUERY_ADDITIONAL_LINKS})
  set_target_properties(libosquery_additional PROPERTIES OUTPUT_NAME osquery_additional)

  # Create the dynamic libosquery_additional.
  if(DEFINED ENV{OSQUERY_BUILD_SHARED})
    add_library(libosquery_additional_shared SHARED ${OSQUERY_ADDITIONAL_SOURCES})
    target_link_libraries(libosquery_additional_shared ${OSQUERY_ADDITIONAL_LINKS})
    SET_OSQUERY_COMPILE(libosquery_additional_shared "${CXX_COMPILE_FLAGS}")
    set_target_properties(libosquery_additional_shared PROPERTIES OUTPUT_NAME osquery_additional)
  endif()

  add_executable(shell devtools/shell.cpp main/shell.cpp)
  if(DEFINED ENV{OSQUERY_BUILD_SHARED})
    target_link_libraries(shell libosquery_shared)
    target_link_libraries(shell libosquery_additional_shared)
  else()
    TARGET_OSQUERY_LINK_WHOLE(shell libosquery)
    TARGET_OSQUERY_LINK_WHOLE(shell libosquery_additional)
  endif()
  SET_OSQUERY_COMPILE(shell "${CXX_COMPILE_FLAGS}")
  set_target_properties(shell PROPERTIES OUTPUT_NAME osqueryi)
  if(WIN32)
    target_link_libraries(shell "linenoise.lib")
  endif()
  add_dependencies(shell osquery_tools)

  add_executable(daemon main/${PROCESS_FAMILY}/daemon.cpp)
  if(DEFINED ENV{OSQUERY_BUILD_SHARED})
    target_link_libraries(daemon libosquery_shared)
    target_link_libraries(daemon libosquery_additional_shared)
  else()
    TARGET_OSQUERY_LINK_WHOLE(daemon libosquery_additional)
    TARGET_OSQUERY_LINK_WHOLE(daemon libosquery)
  endif()
  SET_OSQUERY_COMPILE(daemon "${CXX_COMPILE_FLAGS}")
  set_target_properties(daemon PROPERTIES OUTPUT_NAME osqueryd)

  # Include the public API includes if make devel.
  install(TARGETS libosquery ARCHIVE DESTINATION lib COMPONENT devel OPTIONAL)
  install(DIRECTORY "${CMAKE_SOURCE_DIR}/include/osquery"
    DESTINATION include
    COMPONENT devel OPTIONAL
    PATTERN ".*" EXCLUDE
  )

  # A friendly echo printed after the library is built.
  add_custom_target(osquery_tools ALL
    COMMAND ${CMAKE_COMMAND} -E cmake_echo_color --cyan --bold
      "-- Building osqueryi: ${CMAKE_BINARY_DIR}/osquery/osqueryi"
      "-- Building osqueryd: ${CMAKE_BINARY_DIR}/osquery/osqueryd"
  )

  # make install (executables)
  install(TARGETS shell RUNTIME DESTINATION bin COMPONENT main)
  install(TARGETS daemon RUNTIME DESTINATION bin COMPONENT main)
  install(FILES "${CMAKE_SOURCE_DIR}/tools/deployment/osqueryctl"
    DESTINATION bin COMPONENT main)

  # make install (config files)
  install(FILES "${CMAKE_SOURCE_DIR}/tools/deployment/osquery.example.conf"
    DESTINATION "${CMAKE_INSTALL_PREFIX}/share/osquery/" COMPONENT main)
  install(DIRECTORY DESTINATION "${CMAKE_INSTALL_PREFIX}/var/osquery")

  install(DIRECTORY "${CMAKE_SOURCE_DIR}/packs/"
    DESTINATION "${CMAKE_INSTALL_PREFIX}/share/osquery/packs" COMPONENT main)
  if(APPLE)
    install(FILES "${CMAKE_SOURCE_DIR}/tools/deployment/com.facebook.osqueryd.plist"
      DESTINATION "${CMAKE_INSTALL_PREFIX}/share/osquery/" COMPONENT main)
  else()
    install(PROGRAMS "${CMAKE_SOURCE_DIR}/tools/deployment/osqueryd.initd"
      DESTINATION "/etc/init.d/" RENAME "osqueryd" COMPONENT main)
  endif()
endif()

if(NOT DEFINED ENV{SKIP_TESTS})
  # Detect TR1 usage.
  include(CheckIncludeFileCXX)
  check_include_file_cxx("tr1/tuple" HAVE_TR1_TUPLE)
  if("${HAVE_TR1_TUPLE}" STREQUAL "")
    set(GTEST_FLAGS "-DGTEST_USE_OWN_TR1_TUPLE=1")
  else()
    set(GTEST_FLAGS "-DGTEST_USE_OWN_TR1_TUPLE=0")
  endif()

  set(GTEST_FLAGS
    ${GTEST_FLAGS}
    "-I${CMAKE_SOURCE_DIR}/third-party/gtest-1.7.0/include"
    "-I${CMAKE_SOURCE_DIR}/third-party/gmock-1.7.0/include"
  )
  join("${GTEST_FLAGS}" " " GTEST_FLAGS)

  # osquery testing library (testing helper methods/libs).
  add_library(libosquery_testing STATIC tests/test_util.cpp)
  add_dependencies(libosquery_testing libosquery)
  SET_OSQUERY_COMPILE(libosquery_testing "${CXX_COMPILE_FLAGS}")
  set_target_properties(libosquery_testing PROPERTIES OUTPUT_NAME osquery_testing)

  # osquery core set of unit tests build with SDK.
  add_executable(osquery_tests main/tests.cpp ${OSQUERY_TESTS})
  if(DEFINED ENV{OSQUERY_BUILD_SHARED})
    target_link_libraries(osquery_tests libosquery_shared)
  else()
    TARGET_OSQUERY_LINK_WHOLE(osquery_tests libosquery)
  endif()
  # TODO #2252: Remove this linkage dependency.
  TARGET_OSQUERY_LINK_WHOLE(osquery_tests ${crypto_library})
  target_link_libraries(osquery_tests gtest gmock libosquery_testing)
  SET_OSQUERY_COMPILE(osquery_tests "${GTEST_FLAGS} ${CXX_COMPILE_FLAGS}")
  add_test(osquery_tests osquery_tests)

  # osquery kernel tests.
  if(NOT ${OSQUERY_BUILD_SDK_ONLY} AND NOT WIN32)
    add_executable(osquery_kernel_tests main/tests.cpp ${OSQUERY_KERNEL_TESTS})
    if(DEFINED ENV{OSQUERY_BUILD_SHARED})
      target_link_libraries(osquery_kernel_tests libosquery_shared)
      target_link_libraries(osquery_kernel_tests libosquery_additional_shared)
    else()
      TARGET_OSQUERY_LINK_WHOLE(osquery_kernel_tests libosquery)
      TARGET_OSQUERY_LINK_WHOLE(osquery_kernel_tests libosquery_additional)
    endif()
    target_link_libraries(osquery_kernel_tests gtest gmock libosquery_testing)
    SET_OSQUERY_COMPILE(osquery_kernel_tests "${GTEST_FLAGS} ${CXX_COMPILE_FLAGS} -DKERNEL_TEST=1")
  endif()

  # osquery benchmarks.
  if(NOT DEFINED ENV{SKIP_BENCHMARKS} AND NOT ${OSQUERY_BUILD_SDK_ONLY})
    add_executable(osquery_benchmarks main/benchmarks.cpp ${OSQUERY_BENCHMARKS})
    TARGET_OSQUERY_LINK_WHOLE(osquery_benchmarks libosquery)
    TARGET_OSQUERY_LINK_WHOLE(osquery_benchmarks libosquery_additional)
    target_link_libraries(osquery_benchmarks benchmark libosquery_testing)
    SET_OSQUERY_COMPILE(osquery_benchmarks "${GTEST_FLAGS} ${CXX_COMPILE_FLAGS}")
    set(BENCHMARK_TARGET "$<TARGET_FILE:osquery_benchmarks>")

    # osquery kernel benchmarks.
    add_executable(osquery_kernel_benchmarks main/benchmarks.cpp ${OSQUERY_KERNEL_BENCHMARKS})
    TARGET_OSQUERY_LINK_WHOLE(osquery_kernel_benchmarks libosquery)
    TARGET_OSQUERY_LINK_WHOLE(osquery_kernel_benchmarks libosquery_additional)
    target_link_libraries(osquery_kernel_benchmarks benchmark libosquery_testing)
    SET_OSQUERY_COMPILE(osquery_kernel_benchmarks "${GTEST_FLAGS} ${CXX_COMPILE_FLAGS} -DKERNEL_TEST=1")

    # make benchmark
    add_custom_target(
      run-benchmark
      COMMAND bash -c "${BENCHMARK_TARGET} $ENV{BENCHMARK_TO_FILE}"
      WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
      DEPENDS osquery_benchmarks
    )
  endif()

  if(NOT ${OSQUERY_BUILD_SDK_ONLY})
    # osquery core (additional) set of unit tests built outside of SDK.
    add_executable(osquery_additional_tests main/tests.cpp ${OSQUERY_ADDITIONAL_TESTS})
    if(DEFINED ENV{OSQUERY_BUILD_SHARED})
      target_link_libraries(osquery_additional_tests libosquery_shared)
      target_link_libraries(osquery_additional_tests libosquery_additional_shared)
    else()
      TARGET_OSQUERY_LINK_WHOLE(osquery_additional_tests libosquery)
      TARGET_OSQUERY_LINK_WHOLE(osquery_additional_tests libosquery_additional)
    endif()
    target_link_libraries(osquery_additional_tests gtest gmock libosquery_testing)
    SET_OSQUERY_COMPILE(osquery_additional_tests "${GTEST_FLAGS} ${CXX_COMPILE_FLAGS}")
    add_test(osquery_additional_tests osquery_additional_tests)

    # osquery tables set of unit tests (extracted for organization).
    add_executable(osquery_tables_tests main/tests.cpp ${OSQUERY_TABLES_TESTS})
    if(DEFINED ENV{OSQUERY_BUILD_SHARED})
      target_link_libraries(osquery_tables_tests libosquery_shared)
      target_link_libraries(osquery_tables_tests libosquery_additional_shared)
    else()
      TARGET_OSQUERY_LINK_WHOLE(osquery_tables_tests libosquery)
      TARGET_OSQUERY_LINK_WHOLE(osquery_tables_tests libosquery_additional)
    endif()
    target_link_libraries(osquery_tables_tests gtest gmock libosquery_testing)
    SET_OSQUERY_COMPILE(osquery_tables_tests "${GTEST_FLAGS} ${CXX_COMPILE_FLAGS}")
    add_test(osquery_tables_tests osquery_tables_tests)
  endif()

  # Build the example extension with the SDK.
  ADD_OSQUERY_EXTENSION(example_extension examples/example_extension.cpp)

  # Build the example extension module with the SDK.
  ADD_OSQUERY_MODULE(modexample examples/example_module.cpp)
endif()
