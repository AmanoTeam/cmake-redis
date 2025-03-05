cmake_minimum_required(VERSION 3.19)

project(
	Redis
	VERSION 7.4.0
	DESCRIPTION "Redis is an in-memory database that persists on disk"
	HOMEPAGE_URL "https://github.com/redis/redis"
	LANGUAGES C
)

option(REDIS_ENABLE_LTO "Turn on compiler Link Time Optimizations" OFF)
option(REDIS_ENABLE_TLS "Build Redis with TLS support" OFF)
option(REDIS_ENABLE_SYSTEMD "Build Redis with systemd support" OFF)
option(REDIS_ENABLE_JEMALLOC "Set jemalloc as the memory allocation API" OFF)

set(BUILD_TLS "no")
set(USE_SYSTEMD "no")
set(MALLOC "libc")
set(CFLAGS "${CMAKE_C_FLAGS}")
set(LDFLAGS "")
set(OPT "${CMAKE_C_FLAGS}")

if (REDIS_ENABLE_TLS)
	set(BUILD_TLS "yes")
endif()

if (REDIS_ENABLE_SYSTEMD)
	set(USE_SYSTEMD "yes")
endif()

if (REDIS_ENABLE_JEMALLOC)
	set(MALLOC "jemalloc")
endif()

if (REDIS_ENABLE_LTO)
	set(CFLAGS "${CFLAGS} ${CMAKE_C_COMPILE_OPTIONS_IPO}")
	set(LDFLAGS "${LDFLAGS} ${CMAKE_C_COMPILE_OPTIONS_IPO}")
endif()

string(REPLACE ";" " " CFLAGS "${CFLAGS}")
string(REPLACE ";" " " LDFLAGS "${LDFLAGS}")

if ("${CMAKE_BUILD_TYPE}" STREQUAL "MinSizeRel")
	set(OPT "${OPT} ${CMAKE_C_FLAGS_MINSIZEREL} -Wl,-s")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
	set(OPT "${OPT} ${CMAKE_C_FLAGS_RELEASE} -Wl,-s")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "Debug")
	set(OPT "${OPT} ${CMAKE_C_FLAGS_DEBUG}")
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
	set(OPT "${OPT} ${CMAKE_C_FLAGS_RELWITHDEBINFO}")
endif()

set(REDIS_SOURCE_DIRECTORY "${CMAKE_SOURCE_DIR}/submodules/redis")
set(REDIS_INSTALL_PREFIX "${REDIS_SOURCE_DIRECTORY}/src")

set(REDIS_MAKE_DISTCLEAN "MAKE-DISTCLEAN")

set(REDIS_BENCHMARK "${REDIS_INSTALL_PREFIX}/redis-benchmark")
set(REDIS_CHECK_RDB "${REDIS_INSTALL_PREFIX}/redis-check-rdb")
set(REDIS_SENTINEL "${REDIS_INSTALL_PREFIX}/redis-sentinel")
set(REDIS_CHECK_AOF "${REDIS_INSTALL_PREFIX}/redis-check-aof")
set(REDIS_CLI "${REDIS_INSTALL_PREFIX}/redis-cli")
set(REDIS_SERVER "${REDIS_INSTALL_PREFIX}/redis-server")

set(
	REDIS_TOOLS
	"${REDIS_BENCHMARK}"
	"${REDIS_CHECK_RDB}"
	"${REDIS_SENTINEL}"
	"${REDIS_CHECK_AOF}"
	"${REDIS_CLI}"
	"${REDIS_SERVER}"
)

set(
	REDIS_CONFIGURE_FLAGS
	"BUILD_TLS=${BUILD_TLS}"
	"USE_SYSTEMD=${USE_SYSTEMD}"
	"MALLOC=${MALLOC}"
	"OPT=${OPT}"
	"CFLAGS=${CFLAGS}"
	"LDFLAGS=${LDFLAGS}"
	"uname_S=${CMAKE_SYSTEM_NAME}"
)

list(APPEND REDIS_CONFIGURE_FLAGS "AR=${CMAKE_AR}")

if (CMAKE_CROSSCOMPILING)
	if ("${CMAKE_SYSTEM_NAME}" STREQUAL "Android")
		list(APPEND REDIS_CONFIGURE_FLAGS "CC=${ANDROID_TOOLCHAIN_ROOT}/bin/${TOOLCHAIN_PREFIX}-clang")
	else()
		list(APPEND REDIS_CONFIGURE_FLAGS "CC=${CMAKE_C_COMPILER}")
	endif()
else()
	list(APPEND REDIS_CONFIGURE_FLAGS "CC=${CMAKE_C_COMPILER}")
endif()

add_custom_command(
	OUTPUT "$(REDIS_MAKE_DISTCLEAN)"
	COMMAND make distclean
	WORKING_DIRECTORY "${REDIS_SOURCE_DIRECTORY}"
	COMMENT "-- Cleaning Redis build directory"
)

add_custom_target(
	redis-clean
	DEPENDS "$(REDIS_MAKE_DISTCLEAN)"
)

add_custom_command(
	OUTPUT "${REDIS_SERVER}"
	COMMAND make ${REDIS_CONFIGURE_FLAGS}
	WORKING_DIRECTORY "${REDIS_SOURCE_DIRECTORY}"
	COMMENT "-- Building Redis"
)

add_custom_target(
	redis ALL
	DEPENDS "${REDIS_SERVER}"
)

install(
	FILES ${REDIS_TOOLS}
	PERMISSIONS OWNER_EXECUTE OWNER_WRITE OWNER_READ
	DESTINATION "${CMAKE_INSTALL_PREFIX}/bin"
)
