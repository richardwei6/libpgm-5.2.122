# CMake build script for OpenPGM on Windows

cmake_minimum_required (VERSION 2.8)
project (OpenPGM)

#-----------------------------------------------------------------------------
# force off-tree build

if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
message(FATAL_ERROR "CMake generation is not allowed within the source directory! 
Remove the CMakeCache.txt file and try again from another folder, e.g.: 

   del CMakeCache.txt 
   mkdir cmake-make 
   cd cmake-make
   cmake ..
")
endif(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})

#-----------------------------------------------------------------------------
# dependencies

find_package(PythonInterp REQUIRED)
find_package(Perl REQUIRED)
find_program(PATCH_EXECUTABLE patch)

include (${CMAKE_SOURCE_DIR}/cmake/Modules/TestOpenPGMVersion.cmake)

#-----------------------------------------------------------------------------
# default to Release build

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING
      "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
      FORCE)
endif(NOT CMAKE_BUILD_TYPE)

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin)
set(LIBRARY_OUTPUT_PATH  ${CMAKE_BINARY_DIR}/lib)

#-----------------------------------------------------------------------------
# platform specifics

add_definitions(
	-DWIN32
	-D_CRT_SECURE_NO_WARNINGS
	-DHAVE_FTIME
	-DHAVE_ISO_VARARGS
	-DHAVE_RDTSC
	-DHAVE_WSACMSGHDR
	-DHAVE_DSO_VISIBILITY
	-DUSE_BIND_INADDR_ANY
)

# Parallel make.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")

# Optimization flags.
# http://msdn.microsoft.com/en-us/magazine/cc301698.aspx
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /GL")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} /LTCG")
set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} /LTCG")
set(CMAKE_MODULE_LINKER_FLAGS_RELEASE "${CMAKE_MODULE_LINKER_FLAGS_RELEASE} /LTCG")

#-----------------------------------------------------------------------------
# source files

set(c99-sources
        thread.c
        mem.c
        string.c
        list.c
        slist
        queue.c
        hashtable.c
        messages.c
        error.c
        math.c
        packet_parse.c
        packet_test.c
        sockaddr.c
        time.c
        if.c
	inet_lnaof.c
        getifaddrs.c
	get_nprocs.c
        getnetbyname.c
        getnodeaddr.c
        getprotobyname.c
        indextoaddr.c
        indextoname.c
        nametoindex.c
        inet_network.c
        md5.c
        rand.c
        gsi.c
        tsi.c
        txw.c
        rxw.c
        skbuff.c
        socket.c
        source.c
        receiver.c
        recv.c
        engine.c
        timer.c
        net.c
        rate_control.c
        checksum.c
        reed_solomon.c
        wsastrerror.c
        histogram.c
)

include_directories(
	include
)
set(headers
	include/pgm/atomic.h
	include/pgm/engine.h
	include/pgm/error.h
	include/pgm/gsi.h
	include/pgm/if.h
	include/pgm/in.h
	include/pgm/list.h
	include/pgm/macros.h
	include/pgm/mem.h
	include/pgm/messages.h
	include/pgm/msgv.h
	include/pgm/packet.h
	include/pgm/pgm.h
	include/pgm/skbuff.h
	include/pgm/socket.h
	include/pgm/time.h
	include/pgm/tsi.h
	include/pgm/types.h
	include/pgm/version.h
	include/pgm/winint.h
	include/pgm/wininttypes.h
	include/pgm/zinttypes.h
)

add_definitions(
	-DUSE_16BIT_CHECKSUM
	-DUSE_TICKET_SPINLOCK
	-DUSE_DUMB_RWSPINLOCK
	-DUSE_GALOIS_MUL_LUT
	-DGETTEXT_PACKAGE='"pgm"'
)

#-----------------------------------------------------------------------------
# source generators

foreach (source ${c99-sources})
	string(REGEX REPLACE "\\.c$" "" source "${source}")
	if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${source}.c.c89.patch)
		add_custom_command(
			OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${source}.c89.c
			COMMAND ${CMAKE_COMMAND}
			ARGS	-E
				copy
				${CMAKE_CURRENT_SOURCE_DIR}/${source}.c
				${CMAKE_CURRENT_BINARY_DIR}/${source}.c89.c
			COMMAND ${PATCH_EXECUTABLE}
			ARGS	--binary
				-i
				${CMAKE_CURRENT_SOURCE_DIR}/${source}.c.c89.patch
				${CMAKE_CURRENT_BINARY_DIR}/${source}.c89.c
			DEPENDS ${source}.c
				${CMAKE_CURRENT_SOURCE_DIR}/${source}.c.c89.patch
		)
	else(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${source}.c.c89.patch)
		add_custom_command(
			OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${source}.c89.c
			COMMAND ${CMAKE_COMMAND}
			ARGS	-E
				copy
				${CMAKE_CURRENT_SOURCE_DIR}/${source}.c
				${CMAKE_CURRENT_BINARY_DIR}/${source}.c89.c
			DEPENDS ${source}.c
		)
	endif(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${source}.c.c89.patch)
	list(APPEND generated-results ${CMAKE_CURRENT_BINARY_DIR}/${source}.c89.c)
endforeach()

# generated galois tables
add_custom_command(
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/galois_tables.c89.c
	COMMAND ${PERL_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/galois_generator.pl > ${CMAKE_CURRENT_BINARY_DIR}/galois_tables.c89.c
	DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/galois_generator.pl
)

# version stamping
add_custom_command(
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/version.c89.c
	COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/version_generator.py > ${CMAKE_CURRENT_BINARY_DIR}/version.c89.c
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/version_generator.py
)

set(sources
        ${CMAKE_CURRENT_BINARY_DIR}/galois_tables.c89.c
        ${CMAKE_CURRENT_BINARY_DIR}/version.c89.c
        ${generated-results}
)

#-----------------------------------------------------------------------------
# output

add_library(libpgm STATIC ${sources} ${CMAKE_BINARY_DIR}/NSIS.template.in)
set_target_properties(libpgm PROPERTIES
	RELEASE_POSTFIX "${_pgm_COMPILER}-mt-${OPENPGM_VERSION_MAJOR}_${OPENPGM_VERSION_MINOR}_${OPENPGM_VERSION_MICRO}"
	DEBUG_POSTFIX "${_pgm_COMPILER}-mt-gd-${OPENPGM_VERSION_MAJOR}_${OPENPGM_VERSION_MINOR}_${OPENPGM_VERSION_MICRO}")

#-----------------------------------------------------------------------------
# installer

set(docs
	COPYING
	LICENSE
	README
)
file(GLOB mibs "${CMAKE_CURRENT_SOURCE_DIR}/mibs/*.txt")
set(examples
	examples/async.c
	examples/async.h
	examples/daytime.c
	examples/getopt.c
	examples/getopt.h
	examples/purinrecv.c
	examples/purinsend.c
	examples/shortcakerecv.c
)

if (CMAKE_CL_64)
	set (nsis-template ${CMAKE_SOURCE_DIR}/cmake/NSIS.template64.in)
else (CMAKE_CL_64)
	set (nsis-template ${CMAKE_SOURCE_DIR}/cmake/NSIS.template32.in)
endif (CMAKE_CL_64)
add_custom_command(
	OUTPUT ${CMAKE_BINARY_DIR}/NSIS.template.in
	COMMAND ${CMAKE_COMMAND}
	ARGS    -E
		copy
		${nsis-template}
		${CMAKE_BINARY_DIR}/NSIS.template.in
	DEPENDS ${nsis-template}
)
set (CMAKE_MODULE_PATH "${CMAKE_BINARY_DIR}")

install (TARGETS libpgm DESTINATION lib)
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
	install (FILES ${CMAKE_BINARY_DIR}/lib/libpgm${_pgm_COMPILER}-mt-gd-${OPENPGM_VERSION_MAJOR}_${OPENPGM_VERSION_MINOR}_${OPENPGM_VERSION_MICRO}.pdb DESTINATION lib)
endif (CMAKE_BUILD_TYPE STREQUAL "Debug")
install (FILES ${headers} DESTINATION include/pgm)
foreach (doc ${docs})
	configure_file (${CMAKE_SOURCE_DIR}/${doc} ${CMAKE_BINARY_DIR}/${doc}.txt)
	install (FILES ${CMAKE_BINARY_DIR}/${doc}.txt DESTINATION doc)
endforeach (doc ${docs})
install (FILES ${mibs} DESTINATION mibs)
install (FILES ${examples} DESTINATION examples)

# Only need to ship CRT if distributing executable binaries.
# include (InstallRequiredSystemLibraries)
if (CMAKE_CL_64)
	set (CPACK_NSIS_DISPLAY_NAME "OpenPGM ${OPENPGM_VERSION_MAJOR}.${OPENPGM_VERSION_MINOR}.${OPENPGM_VERSION_MICRO} (x64)")
	set (CPACK_PACKAGE_FILE_NAME "OpenPGM-${OPENPGM_VERSION_MAJOR}.${OPENPGM_VERSION_MINOR}.${OPENPGM_VERSION_MICRO}-x64")
	set (CPACK_INSTALL_CMAKE_PROJECTS
		"${CMAKE_SOURCE_DIR}/build/x64/v110;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug/x64/v110;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/build/x64/v100;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug/x64/v100;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/build/x64/v90;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug/x64/v90;OpenPGM;ALL;/"
	)
else (CMAKE_CL_64)
	set (CPACK_DISPLAY_NAME "OpenPGM ${OPENPGM_VERSION_MAJOR}.${OPENPGM_VERSION_MINOR}.${OPENPGM_VERSION_MICRO}")
	set (CPACK_PACKAGE_FILE_NAME "OpenPGM-${OPENPGM_VERSION_MAJOR}.${OPENPGM_VERSION_MINOR}.${OPENPGM_VERSION_MICRO}-x86")
	set (CPACK_INSTALL_CMAKE_PROJECTS
		"${CMAKE_SOURCE_DIR}/build/x86/v110;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug/x86/v110;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/build/x86/v100;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug/x86/v100;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/build/x86/v90;OpenPGM;ALL;/"
		"${CMAKE_SOURCE_DIR}/debug/x86/v90;OpenPGM;ALL;/"
	)
endif (CMAKE_CL_64)
#set (CPACK_PACKAGE_INSTALL_DIRECTORY "OpenPGM ${CPACK_PACKAGE_VERSION}")
set (CPACK_PACKAGE_VENDOR "Miru")
set (CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
set (CPACK_NSIS_COMPRESSOR "/SOLID lzma")
set (CPACK_PACKAGE_VERSION_MAJOR ${OPENPGM_VERSION_MAJOR})
set (CPACK_PACKAGE_VERSION_MINOR ${OPENPGM_VERSION_MINOR})
set (CPACK_PACKAGE_VERSION_PATCH ${OPENPGM_VERSION_MICRO})
include (CPack)

# end of file
