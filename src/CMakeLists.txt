include_directories(
	"${CMAKE_CURRENT_BINARY_DIR}"
	${PROJECT_SOURCE_DIR}/lib
	${PROJECT_SOURCE_DIR}/src
	${PROJECT_SOURCE_DIR}/src/output_modules
	)

if (APPLE)
	if (EXISTS /opt/local/include)
		include_directories(/opt/local/include)
	endif()
	if (EXISTS /opt/local/lib)
		link_directories(/opt/local/lib)
	endif()
endif()

if (BSD)
	include_directories(/usr/local/include)
	link_directories(/usr/local/lib)
endif()

SET(LIB_SOURCES
	${PROJECT_SOURCE_DIR}/lib/blacklist.c
	${PROJECT_SOURCE_DIR}/lib/constraint.c
	${PROJECT_SOURCE_DIR}/lib/logger.c
	${PROJECT_SOURCE_DIR}/lib/pbm.c
	${PROJECT_SOURCE_DIR}/lib/random.c
	${PROJECT_SOURCE_DIR}/lib/rijndael-alg-fst.c
	${PROJECT_SOURCE_DIR}/lib/xalloc.c
	${PROJECT_SOURCE_DIR}/lib/lockfd.c
	${PROJECT_SOURCE_DIR}/lib/util.c
	${PROJECT_SOURCE_DIR}/lib/queue.c
	${PROJECT_SOURCE_DIR}/lib/csv.c
)

# ADD YOUR PROBE MODULE HERE
SET(EXTRA_PROBE_MODULES
	)

# ADD YOUR OUTPUT MODULE HERE
SET(EXTRA_OUTPUT_MODULES
	)

SET(OUTPUT_MODULE_SOURCES
	output_modules/module_csv.c
	output_modules/output_modules.c
	)

SET(PROBE_MODULE_SOURCES
	probe_modules/module_icmp_echo.c
	probe_modules/module_tcp_synscan.c
	probe_modules/module_udp.c
	probe_modules/module_udp_dns.c
	probe_modules/packet.c
	probe_modules/probe_modules.c
	probe_modules/module_ntp.c
	probe_modules/module_upnp.c
	)

SET(SOURCES
	aesrand.c
	cyclic.c
	expression.c
	fieldset.c
	filter.c
	get_gateway.c
	iterator.c
	monitor.c
	recv.c
	send.c
	shard.c
	socket.c
	state.c
	summary.c
	validate.c
	zmap.c
	zopt_compat.c
	"${CMAKE_CURRENT_BINARY_DIR}/zopt.h"
	"${CMAKE_CURRENT_BINARY_DIR}/lexer.c"
	"${CMAKE_CURRENT_BINARY_DIR}/parser.c"
	${EXTRA_PROBE_MODULES}
	${EXTRA_OUTPUT_MODULES}
	${PROBE_MODULE_SOURCES}
	${OUTPUT_MODULE_SOURCES}
	${LIB_SOURCES}
)

SET(ZBLSOURCES
	zblacklist.c
	${LIB_SOURCES}
)

SET(ZTEESOURCES
  ztee.c
	topt_compat.c
	"${CMAKE_CURRENT_BINARY_DIR}/topt.h"
	${LIB_SOURCES}
)

# Handle various versions of socket
if(WITH_PFRING)
	SET(SOURCES ${SOURCES} socket-pfring.c)
elseif (APPLE OR BSD)
	SET(SOURCES ${SOURCES} socket-bsd.c)
else()
	SET(SOURCES ${SOURCES} socket-linux.c)
endif()

# Handle various versions of recv
if(WITH_PFRING)
	SET(SOURCES ${SOURCES} recv-pfring.c)
else()
	SET(SOURCES ${SOURCES} recv-pcap.c)
endif()

if (WITH_JSON)
	SET(SOURCES ${SOURCES} output_modules/module_json.c)
endif()

if (WITH_REDIS)
	SET(SOURCES ${SOURCES} ${PROJECT_SOURCE_DIR}/lib/redis.c output_modules/module_redis.c output_modules/module_csvredis.c output_modules/module_redis_csv.c)
endif()

add_custom_command(OUTPUT zopt.h
	COMMAND gengetopt -C --no-help --no-version --unamed-opts=SUBNETS -i "${CMAKE_CURRENT_SOURCE_DIR}/zopt.ggo" -F "${CMAKE_CURRENT_BINARY_DIR}/zopt"
	DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/zopt.ggo"
	)

add_custom_command(OUTPUT topt.h
	COMMAND gengetopt -S --no-help --no-version --unamed-opts=FILE -i "${CMAKE_CURRENT_SOURCE_DIR}/topt.ggo" -F "${CMAKE_CURRENT_BINARY_DIR}/topt"
	DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/topt.ggo"
	)

add_custom_command(OUTPUT lexer.c
	COMMAND flex -o "${CMAKE_CURRENT_BINARY_DIR}/lexer.c" --header-file="${CMAKE_CURRENT_BINARY_DIR}/lexer.h" "${CMAKE_CURRENT_SOURCE_DIR}/lexer.l"
	DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/lexer.l"
	)

add_custom_command(OUTPUT parser.c
	COMMAND byacc -d -o "${CMAKE_CURRENT_BINARY_DIR}/parser.c" "${CMAKE_CURRENT_SOURCE_DIR}/parser.y"
	DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/parser.y"
	)

add_executable(zmap ${SOURCES})
add_executable(zblacklist ${ZBLSOURCES})
add_executable(ztee ${ZTEESOURCES})

if (APPLE OR BSD)
	set(DNET_LIBRARIES "dnet")
else()
	set(DNET_LIBRARIES "")
endif()

target_link_libraries(
	zmap
	${PFRING_LIBRARIES}
	pcap gmp m
	${DNET_LIBRARIES}
	${REDIS_LIBS}
	${JSON_LIBRARIES}
	)

target_link_libraries(
	zblacklist
	m
)

target_link_libraries(
	ztee
	m
)

# Install binary
install(
	TARGETS
	zmap
	zblacklist
        ztee
	RUNTIME DESTINATION sbin
)

# Install Manpages
install(
	FILES
	zmap.1
	DESTINATION share/man/man1
	)
