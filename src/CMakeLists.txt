# 3.5 is required for imported boost targets
cmake_minimum_required(VERSION 3.5)

# globally include own includes
include_directories(${PROJECT_SOURCE_DIR}/include)

# read Git revision ID
# WARNING: this value will be stored in the CMake cache
# to update it, you will have to reset the CMake cache
# (doesn't matter for CI builds like Travis for instance, where there's no permanent CMake cache)
execute_process(
    COMMAND git rev-parse --short HEAD
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_COMMIT
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

# set version and build number
set(VERSION 1-alpha)
if(DEFINED ENV{GITHUB_RUN_NUMBER})
    set(BUILD_NUMBER "GitHub actions build $ENV{GITHUB_RUN_NUMBER}")
else()
    set(BUILD_NUMBER "<local dev build>")
endif()

# get current date
execute_process(
    COMMAND env LC_ALL=C date -u "+%Y-%m-%d %H:%M:%S %Z"
    OUTPUT_VARIABLE DATE
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

add_subdirectory(util)
add_subdirectory(plugin)
add_subdirectory(subprocess)
add_subdirectory(core)

add_executable(linuxdeploy main.cpp core.cpp)
target_link_libraries(linuxdeploy linuxdeploy_core args)
set_target_properties(linuxdeploy PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")

target_compile_definitions(linuxdeploy PRIVATE -DLD_GIT_COMMIT="${GIT_COMMIT}")
target_compile_definitions(linuxdeploy PRIVATE -DLD_VERSION="${VERSION}")
target_compile_definitions(linuxdeploy PRIVATE -DLD_BUILD_NUMBER="${BUILD_NUMBER}")
target_compile_definitions(linuxdeploy PRIVATE -DLD_BUILD_DATE="${DATE}")

if(STATIC_BUILD)
    message(WARNING "static builds enabled")
    cmake_minimum_required(VERSION 3.13)
    target_link_options(linuxdeploy PUBLIC -static -static-libgcc -static-libstdc++)
endif()


add_executable(plugin_test plugin_test_main.cpp)
target_link_libraries(plugin_test linuxdeploy_plugin)
set_target_properties(plugin_test PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")

add_executable(appdir_test appdir_test_main.cpp ../include/linuxdeploy/util/assert.h)
target_link_libraries(appdir_test linuxdeploy_core args)
target_include_directories(appdir_test PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/core)
set_target_properties(appdir_test PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")

install(TARGETS linuxdeploy)
