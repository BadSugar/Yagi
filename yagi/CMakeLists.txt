#####################################################
########### Static library use for test #############
#####################################################
project(Yagi)
cmake_minimum_required(VERSION 3.18)

set(yagi_STATIC_SRC
	src/yagiaction.cc
	src/yagiarchitecture.cc
	src/base.cc
	src/exception.cc
	src/ghidra.cc
	src/scope.cc
	src/symbolinfo.cc
	src/typemanager.cc
	src/yagirule.cc
)

set(yagi_STATIC_INCLUDE
	include/yagiaction.hh
	include/yagiarchitecture.hh
	include/base.hh
	include/exception.hh
	include/ghidra.hh
	include/decompiler.hh
	include/loader.hh
	include/logger.hh
	include/scope.hh
	include/symbolinfo.hh
	include/typemanager.hh
	include/typeinfo.hh
	include/yagirule.hh
)

if (APPLE)
    link_directories(${CMAKE_SOURCE_DIR}/ghidra)
    include_directories(${IDASDKDIR})
    include_directories(${IDASDKDIR}/include)
    include_directories(${IDASDKDIR}/lib/x64_mac_clang_64)
    include_directories(${CMAKE_SOURCE_DIR}/ghidra/ghidra/Ghidra/Features/Decompiler/src/decompile/cpp)
    FIND_LIBRARY(LIB_NAME_LIB libida64)
    set(os_ MAC)
    set(IdaSdk ${IDA_SDK_SOURCE_DIR})
    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} ${CMAKE_SOURCE_DIR}/build_yagi)
endif()

add_library(yagi_static STATIC ${yagi_STATIC_INCLUDE} ${yagi_STATIC_SRC})
target_compile_features(yagi_static PRIVATE cxx_std_17)

target_include_directories(
	yagi_static
	INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	PRIVATE ${CMAKE_CURRENT_SOURCE_DIR} ${IDA_SDK_INCLUDE_DIRS} include
)

target_link_libraries(yagi_static libdecomp)

# Yagi source with IDA backend
set(yagi_SRC
	src/ghidradecompiler.cc
	src/yagi.cc
	src/idatype.cc
	src/idalogger.cc
	src/idasymbol.cc
	src/idaloader.cc
	src/plugin.cc
	src/print.cc
	${yagi_STATIC_SRC}
)

set(yagi_INCLUDE
	include/ghidradecompiler.hh
	include/idatype.hh
	include/exception.hh
	include/ghidra.hh
	include/idalogger.hh
	include/idaloader.hh
	include/idasymbol.hh
	include/idatool.hh
	include/plugin.hh
	include/print.hh
	${yagi_STATIC_INCLUDE}
)

# needs IDA SDK for backend
find_package(IdaSdk)

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(MACOSX TRUE)
    set(CMAKE_MACOSX_RPATH 1)
    set(CMAKE_STATIC_LIBRARY_PREFIX "")
    link_directories(${CMAKE_SOURCE_DIR}/ghidra/)
    # The referenced from issue is fixed with this line below
    set(IDA_SDK_LIBS_MACOS "${IDASDKDIR}/lib/x64_mac_clang_64/")
    target_link_libraries(yagi64 PRIVATE libbase.a libdecomp.a libsleigh.a ${IDA_SDK_LIBS_MACOS}/libida64.dylib)
    #target_link_options(yagi PRIVATE /WHOLEARCHIVE:libbase.a)
endif()

if(MSVC)
	add_definitions(
		/wd4267
		/wd4244
		/wd4099
	)
endif()

#####################################################
############### build target yagi64 #################
#####################################################
add_library(yagi64 SHARED ${yagi_INCLUDE} ${yagi_SRC})
target_compile_features(yagi64 PRIVATE cxx_std_17)

target_include_directories(
	yagi64
	INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	PRIVATE ${CMAKE_CURRENT_SOURCE_DIR} ${IDA_SDK_INCLUDE_DIRS} include
)

install(TARGETS yagi64
       RUNTIME DESTINATION plugins
       LIBRARY DESTINATION plugins
)

target_compile_definitions(yagi64 PRIVATE __EA64__)

set_target_properties(yagi64 PROPERTIES 
	OUTPUT_NAME "yagi64"
	PREFIX ""
)

if(MSVC)
	target_compile_definitions(yagi64 PRIVATE __NT__)

	# This is a trick for compiling on visual studio
	# static initializer are stripped because libdecomp are a static library
	# and no PrintLanguage are registred because singleton ctor are never called
	target_link_options(yagi64 PRIVATE /WHOLEARCHIVE:libbase.lib)
endif()

if (not APPLE)
    target_link_libraries(yagi64 libdecomp ${IDA_SDK_LIBS_IDA64})
endif()

#####################################################
# Target for Ida 32, but need to be built in 64 bits#
#####################################################

add_library(yagi SHARED ${yagi_INCLUDE} ${yagi_SRC})

target_compile_features(yagi PRIVATE cxx_std_17)

target_include_directories(
	yagi
	INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	PRIVATE ${CMAKE_CURRENT_SOURCE_DIR} ${IDA_SDK_INCLUDE_DIRS} include
)

install(TARGETS yagi
	RUNTIME DESTINATION plugins
	LIBRARY DESTINATION plugins
)

set_target_properties(yagi PROPERTIES 
	OUTPUT_NAME "yagi"
	PREFIX ""
)

if (${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(MACOSX TRUE)
    set(CMAKE_STATIC_LIBRARY_PREFIX "")
    link_directories(${CMAKE_SOURCE_DIR}/ghidra/)
    set(IDA_SDK_LIBS_MACOS "${IDASDKDIR}/lib/x64_mac_clang_64/")
    target_link_libraries(yagi PRIVATE libbase.a libdecomp.a libsleigh.a ${IDA_SDK_LIBS_MACOS}/libida64.dylib)
endif()

if(MSVC)	
	target_compile_definitions(yagi PRIVATE __NT__)

	# This is a trick for compiling on visual studio
	# static initializer are stripped because libdecomp are a static library
	# and no PrintLanguage are registred because singleton ctor are never called
	target_link_options(yagi PRIVATE /WHOLEARCHIVE:libbase.lib)
endif()

if (not APPLE)
    target_link_libraries(yagi libdecomp.a ${IDA_SDK_LIBS_IDA32})
endif()
