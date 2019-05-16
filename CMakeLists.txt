cmake_minimum_required(VERSION 2.8)

# ppc2 plugin build
set(PROC 			"arm")
set(NAMESPACE		"otawa")
set(ELF_NUM			"40")
set(CONFIG			)

# looking for ARM loader
set(ARM5_PATH "${CMAKE_SOURCE_DIR}/../armv5t")
set(ARM7_PATH "${CMAKE_SOURCE_DIR}/../armv7t")

if(EXISTS "${ARM7_PATH}")
	set(ARM_PATH "${ARM7_PATH}")
	set(ARM7 "ON")
elseif(EXISTS "${ARM5_PATH}")
    set(ARM_PATH "${ARM5_PATH}")
endif()
if(NOT ARM_PATH)
	message(FATAL_ERROR "cannot find any implementation of GLISS ARM")
endif()

find_library(ARM_LIB arm PATHS "${ARM_PATH}/src" NO_DEFAULT_PATH)
if(ARM_LIB)
	message(STATUS "ARM library found at ${ARM_LIB}")
else()
	message(FATAL_ERROR "ARM library not found")
endif()
include_directories("${ARM_PATH}/include")

set(PROC_PREFIX 	"${ARM_PATH}")
set(PROC_LIB 		"${PROC_PREFIX}/src/lib${PROC}.a")
set(PROC_IRG		"${PROC_PREFIX}/${PROC}.irg")

if(ARM7)
    set(ARM_AUX "armv7t.nmp")
else()
    set(ARM_AUX "armv5t.nmp")
endif()


# look for OTAWA
if(NOT OTAWA_CONFIG)
    find_program(OTAWA_CONFIG otawa-config DOC "path to otawa-config")
    if(NOT OTAWA_CONFIG)
        message(FATAL_ERROR "ERROR: otawa-config is required !")
    endif()
endif()
message(STATUS "otawa-config found at ${OTAWA_CONFIG}")


# get OTAWA configuration
execute_process(COMMAND "${OTAWA_CONFIG}" --cflags
	OUTPUT_VARIABLE OTAWA_CFLAGS  OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND "${OTAWA_CONFIG}" --libs
	OUTPUT_VARIABLE OTAWA_LDFLAGS OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND "${OTAWA_CONFIG}" --prefix
	OUTPUT_VARIABLE OTAWA_PREFIX  OUTPUT_STRIP_TRAILING_WHITESPACE)
include("${OTAWA_PREFIX}/share/Otawa/cmake/Otawa.cmake")

add_compile_options(-Wall)
if(CMAKE_VERSION LESS "3.1")
	add_compile_options(--std=c++11)
	message(STATUS "C++11 set using cflags")
else()
	set(CMAKE_CXX_STANDARD 11)
	message(STATUS "C++ set using CMAKE_CXX_STANDARD")
endif()

# sources
file(READ VERSION VERSION)
set(CMAKE_CXX_FLAGS "-Wall -DARM_VERSION=\"${VERSION}\"" )
set(GLISS_ATTR		"${CMAKE_SOURCE_DIR}/../gliss2/gep/gliss-attr")
MAKE_GLISS_FUNCTION(OTAWA_KIND    "otawa_kind"      "otawa_kind.nmp"   		"${PROC_IRG}"   "return 0;" "${ARM_AUX}")
MAKE_GLISS_FUNCTION(OTAWA_TARGET  "otawa_target"    "otawa_target.nmp" 		"${PROC_IRG}"   "return 0;" "${ARM_AUX}")
MAKE_GLISS_PROCEDURE(OTAWA_SEM    "otawa_sem"       "otawa_sem.nmp"    		"${PROC_IRG}"   ";"         "${ARM_AUX}")
MAKE_GLISS_PROCEDURE(OTAWA_KSEM   "otawa_ksem"      "otawa_ksem.nmp"   		"${PROC_IRG}"   ";"         "${ARM_AUX}")
MAKE_GLISS_FUNCTION(OTAWA_MULTI   "otawa_multi"     "otawa_multi.nmp"  		"${PROC_IRG}"   "return 0;"	"${ARM_AUX}")
MAKE_GLISS_FUNCTION(OTAWA_COND    "otawa_condition" "otawa_condition.nmp"   "${PROC_IRG}"   "return 14;"	"${ARM_AUX}")

set(SOURCES
	"arm.cpp"
	${OTAWA_KIND}
	${OTAWA_TARGET}
	${OTAWA_SEM}
	${OTAWA_MULTI}
	${OTAWA_COND}
	${OTAWA_KSEM}
)


# build the library
set(ORIGIN $ORIGIN)
add_library(${PROC} SHARED ${SOURCES})
set_property(TARGET ${PROC} PROPERTY PREFIX "")
set_property(TARGET ${PROC} PROPERTY COMPILE_FLAGS "${OTAWA_CFLAGS}")
target_link_libraries(${PROC} "${OTAWA_LDFLAGS} -Wl,-rpath -Wl,${ORIGIN}/../.. ${PROC_LIB}")


# installation
set(PLUGIN_PATH "${OTAWA_PREFIX}/lib/otawa/${NAMESPACE}")
install(TARGETS ${PROC} LIBRARY		DESTINATION ${PLUGIN_PATH})
install(FILES	${PROC}.eld			DESTINATION ${PLUGIN_PATH})
install(FILES	elf_${ELF_NUM}.eld	DESTINATION ${OTAWA_PREFIX}/lib/otawa/loader)
if(CONFIG)
	install(FILES	${CONFIG}		DESTINATION ${PLUGIN_PATH})
endif()
