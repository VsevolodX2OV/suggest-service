cmake_minimum_required(VERSION 3.12)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(Boost_USE_STATIC_LIBS ON)

option(BUILD_TESTS "Build tests" ON)
option(BUILD_DOCS "Build documentation" OFF)
option(BUILD_COVERAGE "Build code coverage" OFF)

set(
        HUNTER_CACHE_SERVERS
        "https://github.com/bmstu-iu8-cpp-sem-3/hunter-cache"
        CACHE STRING "Default cache server"
)

include("cmake/HunterGate.cmake")

HunterGate(
        URL "https://github.com/cpp-pm/hunter/archive/v0.23.278.tar.gz"
        SHA1 "35fa55bc1dfcf9f4d7264f8bb26cd1759ce1ae07"
)

project(http-server-lab07)
string(APPEND CMAKE_CXX_FLAGS " -pedantic -Werror -Wall -Wextra")
string(APPEND CMAKE_CXX_FLAGS " -Wno-unused-command-line-argument")
string(APPEND CMAKE_CXX_FLAGS " -Wshadow -Wnon-virtual-dtor")

add_compile_definitions(JPATH="${CMAKE_CURRENT_SOURCE_DIR}/suggestions.json")

hunter_add_package(Beast)
find_package(Beast CONFIG REQUIRED)

hunter_add_package(Boost COMPONENTS
        thread
        filesystem
        program_options)
find_package(Boost CONFIG REQUIRED
        thread
        filesystem
        program_options)
find_package(Threads)

hunter_add_package(GTest)
find_package(GTest CONFIG REQUIRED)

hunter_add_package(nlohmann_json)
find_package(nlohmann_json CONFIG REQUIRED)

add_library(${PROJECT_NAME} STATIC
        ${CMAKE_CURRENT_SOURCE_DIR}/sources/suggestion.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/sources/JsonStorage.cpp)

add_executable(demo
        ${CMAKE_CURRENT_SOURCE_DIR}/demo/main.cpp
        )

target_include_directories(${PROJECT_NAME} PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include>"
        )

target_link_libraries(demo ${PROJECT_NAME})
target_link_libraries(${PROJECT_NAME}
        Boost::thread
        Boost::filesystem
        Boost::system
        Beast::Beast
        nlohmann_json::nlohmann_json
        )

if(BUILD_TESTS)
  add_executable(tests
          ${CMAKE_CURRENT_SOURCE_DIR}/tests/test.cpp
          )
  target_link_libraries(tests ${PROJECT_NAME}
          GTest::gtest_main
          GTest::gmock_main)
  enable_testing()
  add_test(NAME unit_tests COMMAND tests)
endif()

if(BUILD_COVERAGE)
  set(ENABLE_COVERAGE ON CACHE BOOL "Enable coverage build." FORCE)
  list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/tools/coverage/cmake")
  find_package(codecov)
  string(APPEND CMAKE_CXX_FLAGS " --coverage")
  add_coverage(${PROJECT_NAME})
  add_coverage(tests)
  coverage_evaluate()
endif()

if(BUILD_DOCS)
  set(DOXYGEN_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/_docs)
  set(DOXYGEN_EXAMPLE_PATH examples)
  set(DOXYGEN_GENERATE_HTML NO)
  set(DOXYGEN_GENERATE_MAN YES)
  find_package(Doxygen)
  doxygen_add_docs(docs include)
  install(DIRECTORY ${DOXYGEN_OUTPUT_DIRECTORY}/man DESTINATION share)
  # add_dependencies(install docs) need to call target docs before install or package
endif()

include(CPackConfig.cmake)

install(TARGETS ${PROJECT_NAME} demo
        EXPORT "${PROJECT_NAME}-targets"
        RUNTIME DESTINATION bin
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        INCLUDES DESTINATION include
        )

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/ DESTINATION include)

install(EXPORT "${PROJECT_NAME}-targets"
        NAMESPACE "${PROJECT_NAME}::"
        DESTINATION "lib/cmake"
        )