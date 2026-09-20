# MIT License
#
# Copyright (c) 2026 The OneLuaPro project authors
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.15)
project(postgres-connector-c C)

# --- Add your cmake subdirectory to the module search path ---
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# --- Include your custom utility module ---
include(PostgresUtils)

# Set C99 standard (required for PCRE2 and some sqlean modules)
set(CMAKE_C_STANDARD 99)
set(CMAKE_VERBOSE_MAKEFILE ON)
if(MSVC)
  # # /wd4244: Conversion from 'type1' to 'type2', possible loss of data
  # # /wd4267: Conversion from 'size_t' to 'type', possible loss of data
  # add_compile_options(/W3 /wd4068 /wd4244 /wd4267)
  if(CMAKE_C_COMPILER_ID MATCHES "IntelLLVM")
    add_compile_options(
      -Wno-deprecated-declarations
      -Wno-inconsistent-dllimport
      # -Wno-unused-variable
      # -Wno-unused-but-set-variable
      # -Wno-unused-function
    )
  endif()
endif()
include (GNUInstallDirs)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})
set(INSTALL_LIBDIR
  ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Installation directory for libraries")
set(INSTALL_BINDIR
  ${CMAKE_INSTALL_BINDIR} CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDEDIR
  ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Installation directory for header files")
set(INSTALL_DOCDIR
  ${CMAKE_INSTALL_DOCDIR} CACHE PATH "Installation directory for documentation")
set(INSTALL_MANDIR
  ${CMAKE_INSTALL_MANDIR} CACHE PATH "Installation directory for manpages")
set(INSTALL_DATAROOTDIR
  ${CMAKE_INSTALL_DATAROOTDIR} CACHE PATH "Installation directory for data")

# --- 1. Versioning & Variables ---
set(POSTGRES_RELEASE "18.6")
string(REPLACE "." ";" VERSION_LIST ${POSTGRES_RELEASE})
list(GET VERSION_LIST 0 MAJOR_VER)
list(GET VERSION_LIST 1 MINOR_VER)
# Calculate numeric version: (MAJOR * 10000) + MINOR
math(EXPR CALCULATED_PG_VERSION_NUM "(${MAJOR_VER} * 10000) + ${MINOR_VER}")

# --- 2. Variablen fuer pg_config.h definieren ---
set(PREFIX "PG_CONN_")

# Feature flags (1 enables the option in the Autotools template)
set(${PREFIX}HAVE_SOCKLEN_T 1)
set(${PREFIX}ENABLE_THREAD_SAFETY 1)
set(${PREFIX}HAVE_ATOMICS 1)
set(${PREFIX}HAVE_STRUCT_ADDRINFO 1)
set(${PREFIX}HAVE_STRUCT_SOCKADDR_STORAGE 1)
set(${PREFIX}HAVE_WINSOCK2_H 1)
set(${PREFIX}HAVE_WS2TCPIP_H 1)
set(${PREFIX}STDC_HEADERS 1)
set(${PREFIX}USE_OPENSSL 1)
set(${PREFIX}HAVE_LIBCRYPTO 1)
set(${PREFIX}HAVE_LIBSSL 1)
set(${PREFIX}HAVE_LONG_INT 1)
set(${PREFIX}HAVE_LONG_LONG_INT 1)
set(${PREFIX}HAVE_INT64_TIMESTAMP 1)

# Core block sizes and modifiers
set(${PREFIX}BLCKSZ 8192)
set(${PREFIX}XLOG_BLCKSZ 8192)
set(${PREFIX}RELSEG_SIZE 131072)
set(${PREFIX}INT64_MODIFIER "\"ll\"")
set(${PREFIX}MEMSET_LOOP_LIMIT 1024)

# Target type sizes for x64 Windows
set(${PREFIX}SIZEOF_SIZE_T 8)
set(${PREFIX}SIZEOF_VOID_P 8)
set(${PREFIX}SIZEOF_LONG_LONG 8)
set(${PREFIX}SIZEOF_LONG 4)
set(${PREFIX}SIZEOF_INT 4)
set(${PREFIX}MAXIMUM_ALIGNOF 8)

# Package metadata
set(${PREFIX}PG_VERSION_NUM ${CALCULATED_PG_VERSION_NUM})
set(${PREFIX}PACKAGE_VERSION "\"${POSTGRES_RELEASE}\"")
set(${PREFIX}PG_VERSION "\"${POSTGRES_RELEASE}\"")
set(${PREFIX}PG_MAJORVERSION "\"${MAJOR_VER}\"")
set(${PREFIX}PG_KRB_SRVNAM "\"postgres\"")
set(${PREFIX}DEF_PGPORT 5432)
set(${PREFIX}DEF_PGPORT_STR "\"5432\"")

# Map PostgreSQL's custom macro directly to the C99 language keyword
set(${PREFIX}pg_restrict "restrict")

# Add this to your variable block (Step 1) in the main project to disable server-side OAuth in client
set(${PREFIX}HAVE_OAUTH 0)

# --- 3. Downloads via FetchContent ---
include(FetchContent)
FetchContent_Declare(
  postgres_source 
  URL "https://ftp.postgresql.org/pub/source/v${POSTGRES_RELEASE}/postgresql-${POSTGRES_RELEASE}.tar.gz"
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)
message(STATUS "Fetching postgresql-${POSTGRES_RELEASE}.tar.gz ...")
FetchContent_MakeAvailable(postgres_source)

# --- 4. File Distribution (Setup internal structure) ---
FetchContent_GetProperties(postgres_source SOURCE_DIR POSTGRES_DIR)



file(READ "${POSTGRES_DIR}/src/include/pg_config.h.in" TEMPLATE_CONTENT)

# Retrieve all currently defined CMake variables
get_cmake_property(ALL_VARIABLES VARIABLES)

foreach(VAR_NAME ${ALL_VARIABLES})
  if(VAR_NAME MATCHES "^${PREFIX}(.+)$")
    set(REAL_VAR "${CMAKE_MATCH_1}") # Extract core variable name (e.g., BLCKSZ)

    # Mirror the prefixed value to a standard variable for configure_file() access
    set(${REAL_VAR} "${${VAR_NAME}}")

    # Robust regex replacement: handles optional preceding newline (?:\r?\n) 
    # and trailing whitespaces/newlines to prevent false-positive matches in comments
    string(REGEX REPLACE 
      "(^|[\r\n])#undef[ \t]+${REAL_VAR}([ \t\r\n]|$)" 
      "\\1#cmakedefine ${REAL_VAR} @${REAL_VAR}@\n" 
      TEMPLATE_CONTENT 
      "${TEMPLATE_CONTENT}"
    )
  endif()
endforeach()


file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/pg_config.h.cmake.in" "${TEMPLATE_CONTENT}")


configure_file(
  "${CMAKE_CURRENT_BINARY_DIR}/pg_config.h.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/pg_config.h"
  @ONLY
)

# Diese Hilfs-Header bleiben als minimale Dummys bestehen
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/pg_config_ext.h
  "#ifndef PG_CONFIG_EXT_H\n#define PG_CONFIG_EXT_H\n#define PG_INT64_TYPE long long\n#endif\n"
)
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/pg_config_paths.h
  "#define PGBINDIR \"${CMAKE_INSTALL_BINDIR}\"\n"
  "#define PGSHAREDIR \"${CMAKE_INSTALL_DATAROOTDIR}/postgresql\"\n"
  "#define SYSCONFDIR \"etc/postgresql\"\n"
  "#define INCLUDEDIR \"${CMAKE_INSTALL_INCLUDEDIR}\"\n"
  "#define PKGINCLUDEDIR \"${CMAKE_INSTALL_INCLUDEDIR}\"\n"
  "#define LOCALEDIR \"${CMAKE_INSTALL_LOCALEDIR}\"\n"
  "#define INCLUDEDIRSERVER \"${CMAKE_INSTALL_INCLUDEDIR}/postgresql/server\"\n"
  "#define LIBDIR \"${CMAKE_INSTALL_LIBDIR}\"\n"
  "#define PKGLIBDIR \"${CMAKE_INSTALL_LIBDIR}/postgresql\"\n"
  "#define DOCDIR \"${CMAKE_INSTALL_DOCDIR}\"\n"
  "#define HTMLDIR \"${CMAKE_INSTALL_DOCDIR}/html\"\n"
  "#define MANDIR \"${CMAKE_INSTALL_MANDIR}\"\n"
)
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/pg_config_os.h
  "#include \"port/win32.h\"\n"
)

# --- Generate the real utils/errcodes.h using the included function ---
set(ERRCODES_TXT "${POSTGRES_DIR}/src/backend/utils/errcodes.txt")
set(ERRCODES_H   "${CMAKE_CURRENT_BINARY_DIR}/utils/errcodes.h")

# Ensure the output subdirectory exists within the build tree
file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/utils")

# Execute the isolated function
generate_postgres_errcodes("${ERRCODES_TXT}" "${ERRCODES_H}")

# --- set core sources
set(LIBPQ_SOURCES
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-auth.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-connect.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-exec.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-misc.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-print.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-protocol3.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-secure.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-secure-openssl.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-secure-common.c
  ${POSTGRES_DIR}/src/interfaces/libpq/libpq-events.c
  ${POSTGRES_DIR}/src/interfaces/libpq/pqexpbuffer.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-auth-oauth.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-auth-scram.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-trace.c
  ${POSTGRES_DIR}/src/interfaces/libpq/fe-cancel.c
  ${POSTGRES_DIR}/src/interfaces/libpq/pthread-win32.c
  ${POSTGRES_DIR}/src/interfaces/libpq/win32.c

  # WIN32-specific / Portability layer aus src/common
  ${POSTGRES_DIR}/src/common/string.c
  ${POSTGRES_DIR}/src/common/link-canary.c
  ${POSTGRES_DIR}/src/common/encnames.c
  ${POSTGRES_DIR}/src/common/pg_prng.c
  ${POSTGRES_DIR}/src/common/scram-common.c
  ${POSTGRES_DIR}/src/common/hmac_openssl.c
  ${POSTGRES_DIR}/src/common/base64.c
  ${POSTGRES_DIR}/src/common/fe_memutils.c
  ${POSTGRES_DIR}/src/common/ip.c
  ${POSTGRES_DIR}/src/common/wchar.c
  ${POSTGRES_DIR}/src/common/jsonapi.c
  ${POSTGRES_DIR}/src/common/cryptohash.c
  ${POSTGRES_DIR}/src/common/cryptohash_openssl.c
  ${POSTGRES_DIR}/src/common/md5_common.c
  ${POSTGRES_DIR}/src/common/md5.c
  ${POSTGRES_DIR}/src/common/saslprep.c
  ${POSTGRES_DIR}/src/common/stringinfo.c
  ${POSTGRES_DIR}/src/common/unicode_norm.c
  ${POSTGRES_DIR}/src/common/psprintf.c

  # WIN32-specific / Portability layer aus src/port
  ${POSTGRES_DIR}/src/port/path.c
  ${POSTGRES_DIR}/src/port/pgstrcasecmp.c
  ${POSTGRES_DIR}/src/port/chklocale.c
  ${POSTGRES_DIR}/src/port/strerror.c
  ${POSTGRES_DIR}/src/port/snprintf.c
  ${POSTGRES_DIR}/src/port/pg_strong_random.c
  ${POSTGRES_DIR}/src/port/win32gettimeofday.c
  ${POSTGRES_DIR}/src/port/win32common.c
  ${POSTGRES_DIR}/src/port/strlcpy.c
  ${POSTGRES_DIR}/src/port/strlcat.c
  ${POSTGRES_DIR}/src/port/explicit_bzero.c
  ${POSTGRES_DIR}/src/port/open.c
  ${POSTGRES_DIR}/src/port/win32stat.c
  ${POSTGRES_DIR}/src/port/win32error.c
  ${POSTGRES_DIR}/src/port/noblock.c
  ${POSTGRES_DIR}/src/port/inet_net_ntop.c
  ${POSTGRES_DIR}/src/port/win32gai_strerror.c
  ${POSTGRES_DIR}/src/port/getpeereid.c
  ${POSTGRES_DIR}/src/port/win32setlocale.c
  ${POSTGRES_DIR}/src/port/win32ntdll.c
  ${POSTGRES_DIR}/src/port/win32link.c
  ${POSTGRES_DIR}/src/port/pgsleep.c
  ${POSTGRES_DIR}/src/port/inet_aton.c
  ${POSTGRES_DIR}/src/port/timingsafe_bcmp.c
  ${POSTGRES_DIR}/src/port/dirmod.c
)

add_library(pq STATIC ${LIBPQ_SOURCES})

target_include_directories(pq PRIVATE
  ${LUA_HINTS}/include
  ${POSTGRES_DIR}/src/include/port/win32
  ${POSTGRES_DIR}/src/include/port/win32_msvc
  ${POSTGRES_DIR}/src/port
)

target_include_directories(pq PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<BUILD_INTERFACE:${POSTGRES_DIR}/src/include>
  $<BUILD_INTERFACE:${POSTGRES_DIR}/src/interfaces/libpq>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# 'WIN32' aktiviert Windows-Sockets; 'NOT_USED' deaktiviert SSL (fuer minimale Abhaengigkeiten)
target_compile_definitions(pq PRIVATE
  WIN32
  _WINDOWS
  FD_SETSIZE=1024
  _CRT_SECURE_NO_WARNINGS
  #_WINSOCK_DEPRECATED_NO_WARNINGS
  FRONTEND
  #WIN32_ONLY_COMPILER
)

target_link_libraries(pq INTERFACE
  ${LUA_HINTS}/lib/ssl.lib
  ${LUA_HINTS}/lib/crypto.lib
  ws2_32
  secur32
  advapi32
  shell32
)

# Write the actual targets to an export set named "PostgresConnectorTargets"
install(TARGETS pq
  EXPORT PostgresConnectorTargets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

# Install the standard public header files to /include
install(FILES 
  ${POSTGRES_DIR}/src/interfaces/libpq/libpq-fe.h
  ${POSTGRES_DIR}/src/interfaces/libpq/libpq-events.h
  ${POSTGRES_DIR}/src/include/postgres_ext.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Generate and install the named Target File (PostgresConnectorTargets.cmake)
install(EXPORT PostgresConnectorTargets
  FILE PostgresConnectorTargets.cmake
  NAMESPACE PostgresConnector::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/PostgresConnector
)

# Create the main Package Config File dynamically in the binary directory
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/PostgresConnectorConfig.cmake"
  "include(CMakeFindDependencyMacro)\n"
  "# Include the auto-generated targets file\n"
  "include(\"\${CMAKE_CURRENT_LIST_DIR}/PostgresConnectorTargets.cmake\")\n"
)

# Install the Package Config File so find_package() can discover it
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/PostgresConnectorConfig.cmake"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/PostgresConnector
)
