###########################################################################
#
# Copyright 2017 Realm Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
###########################################################################
cmake_minimum_required(VERSION 3.21.4)
project(RealmJava)

# For debugging: Enable this to output all variables defined for the project
#get_cmake_property(_variableNames VARIABLES)
#foreach (_variableName ${_variableNames})
#    message(STATUS "${_variableName}=${${_variableName}}")
#endforeach()

# loading dependencies properties
file(STRINGS "${CMAKE_SOURCE_DIR}/../../../../../dependencies.list" DEPENDENCIES)
foreach(LINE IN LISTS DEPENDENCIES)

    string(REGEX MATCHALL "([^=]+)" KEY_VALUE "${LINE}")
    list(LENGTH KEY_VALUE matches_count)
    if(matches_count STREQUAL 2)
        list(GET KEY_VALUE 0 KEY)
        list(GET KEY_VALUE 1 VALUE)
        set(DEP_${KEY} ${VALUE})
    endif()
endforeach()

FUNCTION(capitalizeFirstLetter var value)
    string(SUBSTRING ${value} 0 1 firstLetter)
    string(TOUPPER ${firstLetter} firstLetter)
    string(REGEX REPLACE "^.(.*)" "${firstLetter}\\1" value "${value}")
    set(${var} "${value}" PARENT_SCOPE)
ENDFUNCTION(capitalizeFirstLetter)

# find javah
find_package(Java COMPONENTS Development)
if (NOT Java_Development_FOUND)
    if (DEFINED ENV{JAVA_HOME} AND EXISTS "$ENV{JAVA_HOME}/bin/javah")
        set(Java_JAVAH_EXECUTABLE "$ENV{JAVA_HOME}/bin/javah")
    elseif (EXISTS "/usr/bin/javah")
        set(Java_JAVAH_EXECUTABLE "/usr/bin/javah")
    else()
        message(FATAL_ERROR "Cannot find javah")
    endif()
endif()
include (UseJava)

set(CMAKE_VERBOSE_MAKEFILE ON)
# Generate compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Initialize common compile & link flags.
set(REALM_LINKER_FLAGS "")
set(REALM_COMMON_CXX_FLAGS "-DREALM_PLATFORM_JAVA=1")

# Setup lcache
if(NDK_LCACHE)
    set(CMAKE_CXX_CREATE_SHARED_LIBRARY "${NDK_LCACHE} ${CMAKE_CXX_CREATE_SHARED_LIBRARY}")
endif()

# Set option flags for Core.
# See https://github.com/realm/realm-core/blob/master/CMakeLists.txt#L174 for the full list.
if (REALM_FLAVOR STREQUAL base)
    set(REALM_ENABLE_SYNC OFF)
else()
    set(REALM_ENABLE_SYNC ON)
endif()

# Format strings used to represent build parameters: Variant and Type
string(TOLOWER ${CMAKE_BUILD_TYPE} build_type_FOLDER)
set(realmFlavorCap "")
set(buildTypeCap "")
capitalizeFirstLetter(realmFlavorCap "${REALM_FLAVOR}")
capitalizeFirstLetter(buildTypeCap "${CMAKE_BUILD_TYPE}")
message("Build type: ${CMAKE_BUILD_TYPE}")
if(CMAKE_BUILD_TYPE STREQUAL RelWithDebInfo)
    # From NDK 23, Android uses CMake "RelWithDebInfo" instead of "Release", but we create the JNI
    # headers from Gradle using the `debug`/`release` variant names, so rename
    set(buildTypeCap "Release")
endif()

# The JNI header files are generated by the Java compile task.
message("JNI Header directory: ${CMAKE_SOURCE_DIR}/../../../build/intermediates/javac/${REALM_FLAVOR}${buildTypeCap}/jni_include")
set(jni_headers_PATH ${CMAKE_SOURCE_DIR}/../../../build/intermediates/javac/${REALM_FLAVOR}${buildTypeCap}/jni_include)

# Check if user defined download location
if(NOT DEFINED ENV{REALM_CORE_DOWNLOAD_DIR})
    set(REALM_CORE_DOWNLOAD_DIR ${CMAKE_BINARY_DIR})
else()
    set(REALM_CORE_DOWNLOAD_DIR $ENV{REALM_CORE_DOWNLOAD_DIR})
endif()

if(REALM_JAVA_BUILD_CORE_FROM_SOURCE OR NOT REALM_ENABLE_SYNC)
    message(STATUS "Building Realm Core from source...")

    set(REALM_BUILD_LIB_ONLY ON)
    add_subdirectory(realm-core EXCLUDE_FROM_ALL)
else()
    message(STATUS "Using prebuilt Realm Core.")

    # CMAKE_INTERPROCEDURAL_OPTIMIZATION is not compatible with binary artifacts.
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION OFF)

    execute_process(COMMAND "${CMAKE_COMMAND}" "-E" "environment")
    set(core_filename "realm-${CMAKE_BUILD_TYPE}-v${DEP_REALM_CORE}-Android-${CMAKE_ANDROID_ARCH_ABI}-devel")

    # Use cached resources if possible
    if(NOT EXISTS "${REALM_CORE_DOWNLOAD_DIR}/${core_filename}.tar.gz")
        message(STATUS "Downloading ${core_filename}...")
        file(DOWNLOAD "https://static.realm.io/downloads/core/${core_filename}.tar.gz" "${REALM_CORE_DOWNLOAD_DIR}/${core_filename}.tar.gz")
    else()
        file(TIMESTAMP "${REALM_CORE_DOWNLOAD_DIR}/${core_filename}.tar.gz" CORE_FILE_TIMESTAMP "%b %d %H:%M")
        message(STATUS "Using cached filename: ${core_filename}.tar.gz : ${CORE_FILE_TIMESTAMP}")
    endif()

    message(STATUS "Uncompressing realm-core...")
    file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/${core_filename}")

    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xfz "${REALM_CORE_DOWNLOAD_DIR}/${core_filename}.tar.gz"
            WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/${core_filename}")

    include(${CMAKE_BINARY_DIR}/${core_filename}/lib/cmake/Realm/RealmConfig.cmake)
endif()

# build application's shared lib
include_directories(
    ${CMAKE_SOURCE_DIR}
    ${jni_headers_PATH}
)

# Hack the memmove bug on Samsung device.
if (ARMEABI OR ARMEABI_V7A)
    set(REALM_LINKER_FLAGS "${REALM_LINKER_FLAGS} -Wl,--wrap,memmove -Wl,--wrap,memcpy")
    set(REALM_COMMON_CXX_FLAGS "${REALM_COMMON_CXX_FLAGS} -DREALM_WRAP_MEMMOVE=1")
else()
    set(REALM_COMMON_CXX_FLAGS "${REALM_COMMON_CXX_FLAGS} -DREALM_WRAP_MEMMOVE=0")
endif()

#FIXME uninitialized is reported by query_expression.hpp:1070
#      d.init(ValueBase::m_from_link_list, ValueBase::m_values, D{});
#FIXME maybe-uninitialized is reported by table_view.cpp:272:15:
#     'best.m_nanoseconds' was declared here
#     -Wno-missing-field-initializers disable in object store as well.
# FIXME See https://github.com/realm/realm-core/issues/4732
set(WARNING_CXX_FLAGS "-Werror -Wall -Wextra -pedantic -Wmissing-declarations \
    -Wempty-body -Wparentheses -Wunknown-pragmas -Wunreachable-code \
    -Wno-missing-field-initializers -Wno-unevaluated-expression -Wno-unreachable-code \
    -Wno-c99-extensions")
set(REALM_COMMON_CXX_FLAGS "${REALM_COMMON_CXX_FLAGS} -DREALM_ANDROID -DREALM_HAVE_CONFIG -DPIC -fdata-sections -pthread -frtti -fvisibility=hidden -fsigned-char -fno-stack-protector -std=c++17")
if (REALM_ENABLE_SYNC)
    set(REALM_COMMON_CXX_FLAGS "${REALM_COMMON_CXX_FLAGS} -DREALM_ENABLE_SYNC=1")
endif()
set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG -O2")
# -ggdb doesn't play well with -flto
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -glldb -g")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${REALM_COMMON_CXX_FLAGS} ${WARNING_CXX_FLAGS} ${ABI_CXX_FLAGS}")

# Set Linker flags flags
if (CMAKE_BUILD_TYPE STREQUAL "Release")
    set(REALM_LINKER_FLAGS "${REALM_LINKER_FLAGS} -Wl,-gc-sections -Wl,--exclude-libs,ALL")
endif()
if (REALM_ENABLE_SYNC)
    set(REALM_LINKER_FLAGS "${REALM_LINKER_FLAGS} -lz")
endif()
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${REALM_LINKER_FLAGS}")

# JNI source files
file(GLOB jni_SRC
    "*.cpp"
    "jni_util/*.cpp"
    "jni_impl/android_logger.cpp"
)
# Those source file are only needed for sync.
if (NOT REALM_ENABLE_SYNC)
    list(REMOVE_ITEM jni_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_User.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_FunctionsImpl.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_EmailPasswordAuthImpl.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_ApiKeyAuthImpl.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_sync_ClientResetRequiredError.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_sync_Sync.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_sync_SyncSession.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsApp.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsAsyncOpenTask.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsAppCredentials.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsJavaNetworkTransport.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsMongoClient.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsMongoCollection.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsWatchStream.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsMongoDatabase.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsMutableSubscriptionSet.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsSubscription.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsSubscriptionSet.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsPush.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_internal_objectstore_OsSyncUser.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_mongo_iterable_AggregateIterable.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/io_realm_mongodb_mongo_iterable_FindIterable.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/jni_util/bson_util.cpp
    )
endif()

add_library(realm-jni SHARED ${jni_SRC})
target_link_libraries(realm-jni log android Realm::ObjectStore)

# Strip the release so files and backup the unstripped versions
if (buildTypeCap STREQUAL "Release")
    set(unstripped_SO_DIR
        "${CMAKE_SOURCE_DIR}/../../../build/outputs/jniLibs-unstripped/${REALM_FLAVOR}/${ANDROID_ABI}")
    add_custom_command(TARGET realm-jni
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${unstripped_SO_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:realm-jni> ${unstripped_SO_DIR}
        COMMAND ${CMAKE_STRIP} $<TARGET_FILE:realm-jni>)
endif()
