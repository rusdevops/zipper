###############################################################################
#
# Description       : CMake build script for zipper
# Original author(s): Frank Bergmann <fbergman@caltech.edu>
# Organization      : California Institute of Technology
#
###############################################################################

set(CMAKE_LEGACY_CYGWIN_WIN32 0)
cmake_minimum_required(VERSION 3.3)

include(cmake/HunterGate.cmake)
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.19.109.tar.gz"
    SHA1 "2c416fea5f66429ffb4412ce5b8d0c6799ec44d2"
)

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION.txt")

    file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/VERSION.txt" VersionString NEWLINE_CONSUME)
    string(STRIP "${VersionString}" VersionString)
    set(ZIPPER_DOTTED_VERSION ${VersionString})
    string(REPLACE "." ";" VersionString "${VersionString}" )
    string(REPLACE "-" ";" VersionString "${VersionString}" )
    list(LENGTH VersionString versionLength)
    list(GET VersionString 0 ZIPPER_VERSION_MAJOR )
    list(GET VersionString 1 ZIPPER_VERSION_MINOR )
    list(GET VersionString 2 ZIPPER_VERSION_PATCH )

    if(${versionLength} GREATER 3)
        list(GET VersionString 3 ZIPPER_VERSION_RELEASE )
    endif()

endif()

set(ZIPPER_VERSION ${ZIPPER_VERSION_MAJOR}.${ZIPPER_VERSION_MINOR}.${ZIPPER_VERSION_PATCH})
project(zipper VERSION ${ZIPPER_VERSION})

include (CMakeTestCCompiler)
include (CheckCSourceCompiles)
include (CheckCXXSourceCompiles)
include (CheckStructHasMember)
include (CheckLibraryExists)
include (CheckFunctionExists)
include (CheckCCompilerFlag)
include (CheckCSourceRuns)
include (CheckSymbolExists)
include (CheckTypeSize)

# version number needs to be calculated correctly
MATH(EXPR ZIPPER_VERSION_NUMERIC "${ZIPPER_VERSION_MAJOR} * 10000 + ${ZIPPER_VERSION_MINOR} * 100 + ${ZIPPER_VERSION_PATCH}" )
set(PACKAGE_VERSION "${ZIPPER_VERSION_MAJOR}.${ZIPPER_VERSION_MINOR}.${ZIPPER_VERSION_PATCH}${ZIPPER_VERSION_RELEASE}")
set(PACKAGE_NAME "Zipper")

# add make dist and make check target as they are already familiar for
# everyone using the gnumake build
add_custom_target(dist COMMAND ${CMAKE_MAKE_PROGRAM} package_source)
add_custom_target(check COMMAND ${CMAKE_MAKE_PROGRAM} test)

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "C++ wrapper around minizip compression library.")
set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "The SBML Team")
set(CPACK_PACKAGE_CONTACT "SBML Team <sbml-team@caltech.edu>, Sebastian <devsebas@gmail.com>")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md")
set(CPACK_PACKAGE_VERSION_MAJOR "${ZIPPER_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${ZIPPER_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${ZIPPER_VERSION_PATCH}")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "Libraries/Development")
set(CPACK_DEBIAN_PACKAGE_SECTION "Libraries")

set(CPACK_SOURCE_IGNORE_FILES "${CMAKE_CURRENT_BINARY_DIR};/.svn/;/.libs/;/.deps/;/.bzr/;.*.o$;.*.lo$;.*.la$;/.git/;${CPACK_SOURCE_IGNORE_FILES};/.DS_Store;/.svnignore;blib;libsbml-dist;*.txt.user")

set(ADDITIONAL_LIB_DIRS)
if(CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "i386")
    set(CPACK_RPM_PACKAGE_ARCHITECTURE "i386")
else()
    set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "amd64")
    set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")
    set(ADDITIONAL_LIB_DIRS "/usr/lib64" "/usr/lib/x86_64-linux-gnu/")
endif()

include(CPack)

set(ZIPPER_LIBRARY zipper)
set(ZIPPER_NAMESPACE "zipper::")

set(USE_ZLIB ON)
add_definitions( -DUSE_ZLIB )

include_directories(BEFORE ${CMAKE_CURRENT_SOURCE_DIR}/include/)
include_directories(BEFORE ${CMAKE_CURRENT_SOURCE_DIR}/include/zipper)

hunter_add_package(ZLIB)
find_package(ZLIB CONFIG REQUIRED)

hunter_add_package(minizip)
find_package(minizip CONFIG REQUIRED)

file(GLOB ${PROJECT_NAME}_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/zipper/*.cpp")

add_library (${ZIPPER_LIBRARY} STATIC ${${PROJECT_NAME}_SOURCES})
  
target_link_libraries(${ZIPPER_LIBRARY} ZLIB::zlib minizip::minizip)

target_include_directories(${ZIPPER_LIBRARY} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

set(GENERATED_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(CONFIG_INSTALL_DIR "lib/cmake/${PROJECT_NAME}")
set(INCLUDE_INSTALL_DIR "include")

set(VERSION_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}ConfigVersion.cmake")
set(PROJECT_CONFIG "${GENERATED_DIR}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${VERSION_CONFIG}" COMPATIBILITY SameMajorVersion
)


configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in"
    "${PROJECT_CONFIG}"
     INSTALL_DESTINATION "${CONFIG_INSTALL_DIR}"
)
  
install(TARGETS ${ZIPPER_LIBRARY}
		  EXPORT "${TARGETS_EXPORT_NAME}"
          RUNTIME DESTINATION bin
          LIBRARY DESTINATION lib
          ARCHIVE DESTINATION lib
)

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/ DESTINATION include)

install(
  FILES "${PROJECT_CONFIG}" "${VERSION_CONFIG}"
  DESTINATION "${CONFIG_INSTALL_DIR}"
  )


install(
	EXPORT "${TARGETS_EXPORT_NAME}" 
    NAMESPACE "${ZIPPER_NAMESPACE}" 
    DESTINATION "${CONFIG_INSTALL_DIR}"
)



option(BUILD_TEST "Build the test program." ON)
if (BUILD_TEST)
  enable_testing()

  file(GLOB TEST_SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/test/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/test/*.hpp
  )

  if (UNIX)
        # setup valgrind

        set(CMAKE_MEMORYCHECK_COMMAND valgrind)
        set(CMAKE_MEMORYCHECK_COMMAND_OPTIONS
           "--error-exitcode=1 --trace-children=yes --leak-check=full --show-reachable=yes --leak-resolution=high --track-origins=yes --error-limit=no ")

        set(MEMCHECK_COMMAND
           "${CMAKE_MEMORYCHECK_COMMAND} ${CMAKE_MEMORYCHECK_COMMAND_OPTIONS}")
        separate_arguments(MEMCHECK_COMMAND)
  endif()

  include(CTest)

  add_executable(${ZIPPER_LIBRARY}-test ${TEST_SOURCES})
  target_link_libraries(${ZIPPER_LIBRARY}-test  zipper ZLIB::zlib minizip::minizip)

  add_test(
    NAME test_zipper_run
    COMMAND $<TARGET_FILE:${ZIPPER_LIBRARY}-test>
  )
  
  set_tests_properties(test_zipper_run 
    PROPERTIES PASS_REGULAR_EXPRESSION "All tests passed"
  )
endif()

message(STATUS "
----------------------------------------------------------------------
Zipper version ${PACKAGE_VERSION}
----------------------------------------------------------------------

   Configured on host '${HOSTNAME}'
     host type                       = ${CMAKE_SYSTEM_NAME}
     host operating system           = ${CMAKE_SYSTEM_NAME} ${CMAKE_SYSTEM_VERSION}
     host CPU architecture           = ${CMAKE_SYSTEM_PROCESSOR}

   General build flags:
     CC                              = ${CMAKE_C_COMPILER}
     CXX                             = ${CMAKE_CXX_COMPILER}
     CPPFLAGS                        = ${BUILD_DEFINITIONS}
     CFLAGS                          = ${CMAKE_C_FLAGS}
     CXXFLAGS                        = ${CMAKE_CXX_FLAGS}
     LDFLAGS                         = ${CMAKE_EXE_LINKER_FLAGS}

   Zlib library configuration:
     Zlib library                    = ${LIBZ_LIBRARY}
     Zlib include dir                = ${LIBZ_INCLUDE_DIR}

   Other configuration settings:
     Installation $prefix            = ${CMAKE_INSTALL_PREFIX}
     ")

