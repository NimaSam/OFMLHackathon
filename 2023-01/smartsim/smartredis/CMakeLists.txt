# BSD 2-Clause License
#
# Copyright (c) 2021-2023, Hewlett Packard Enterprise
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# 1. Redistributions of source code must retain the above copyright notice, this
#    list of conditions and the following disclaimer.
#
# 2. Redistributions in binary form must reproduce the above copyright notice,
#    this list of conditions and the following disclaimer in the documentation
#    and/or other materials provided with the distribution.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

project(SmartRedis)

cmake_minimum_required(VERSION 3.13)

option(BUILD_PYTHON  "Build the python module" ON)
option(BUILD_FORTRAN "Build the fortran client library" OFF)

set(CMAKE_BUILD_TYPE RELEASE)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_INSTALL_PREFIX ${CMAKE_SOURCE_DIR}/install)
set(CMAKE_CXX_VISIBILITY_PRESET default)
set(THREADS_PREFER_PTHREAD_FLAG ON)

if (BUILD_FORTRAN)
    enable_language(Fortran)
endif()

if (WERROR)
    if (CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
        add_compile_options(-Wall -Werror)
    else()
        message(WARNING "WERROR was specified, but the CMAKE compiler is not GCC")
    endif()
endif()

find_library(REDISPP redis++ PATHS ${CMAKE_SOURCE_DIR}/install/lib NO_DEFAULT_PATH REQUIRED)
find_library(HIREDIS hiredis PATHS ${CMAKE_SOURCE_DIR}/install/lib NO_DEFAULT_PATH REQUIRED)
find_package(Threads REQUIRED)

set(EXT_CLIENT_LIBRARIES ${REDISPP} ${HIREDIS})

set(CLIENT_SRC
    src/c/c_client.cpp
    src/c/c_dataset.cpp
    src/c/c_error.cpp
    src/c/c_logcontext.cpp
    src/c/c_logger.cpp
    src/cpp/address.cpp
    src/cpp/client.cpp
    src/cpp/dataset.cpp
    src/cpp/command.cpp
    src/cpp/keyedcommand.cpp
    src/cpp/nonkeyedcommand.cpp
    src/cpp/multikeycommand.cpp
    src/cpp/singlekeycommand.cpp
    src/cpp/compoundcommand.cpp
    src/cpp/addressatcommand.cpp
    src/cpp/addressanycommand.cpp
    src/cpp/addressallcommand.cpp
    src/cpp/clusterinfocommand.cpp
    src/cpp/dbinfocommand.cpp
    src/cpp/gettensorcommand.cpp
    src/cpp/commandlist.cpp
    src/cpp/metadata.cpp
    src/cpp/tensorbase.cpp
    src/cpp/tensorpack.cpp
    src/cpp/dbnode.cpp
    src/cpp/commandreply.cpp
    src/cpp/redisserver.cpp
    src/cpp/rediscluster.cpp
    src/cpp/redis.cpp
    src/cpp/metadatafield.cpp
    src/cpp/stringfield.cpp
    src/cpp/pipelinereply.cpp
    src/cpp/threadpool.cpp
    src/cpp/utility.cpp
    src/cpp/logger.cpp
    src/cpp/srobject.cpp
)

include_directories(SYSTEM
    include
    install/include
)

if (BUILD_FORTRAN)
    set(FORTRAN_SRC
        src/fortran/client.F90
        src/fortran/dataset.F90
        src/fortran/errors.F90
        src/fortran/fortran_c_interop.F90
        src/fortran/logcontext.F90
        src/fortran/logger.F90
    )
    include_directories(src/fortran)
    # Note the following has to be before ANY add_library command)
    set(CMAKE_Fortran_MODULE_DIRECTORY "${CMAKE_INSTALL_PREFIX}/include")
    # Fortran library
    add_library(smartredis-fortran SHARED ${FORTRAN_SRC})
    set_target_properties(smartredis-fortran PROPERTIES SUFFIX ".so")
    target_link_libraries(smartredis-fortran PUBLIC smartredis ${EXT_CLIENT_LIBRARIES})
    # Install dynamic library and headers
    install(TARGETS smartredis-fortran
    	LIBRARY DESTINATION lib)
endif()


# Build dynamic library
add_library(smartredis SHARED ${CLIENT_SRC})
set_target_properties(smartredis PROPERTIES SUFFIX ".so")
target_link_libraries(smartredis PUBLIC ${EXT_CLIENT_LIBRARIES} PRIVATE Threads::Threads)

install(DIRECTORY "${CMAKE_SOURCE_DIR}/include/"
        DESTINATION "include"
        FILES_MATCHING
        PATTERN "*.h" PATTERN "*.tcc" PATTERN "*.inc"
)

# Install dynamic library and headers
install(TARGETS smartredis
     	LIBRARY DESTINATION lib)

if(BUILD_PYTHON)
	message("-- Python client build enabled")
	add_subdirectory(${CMAKE_SOURCE_DIR}/third-party/pybind
                     ${CMAKE_SOURCE_DIR}/third-party/pybind/build)

    add_library(smartredis_static STATIC ${CLIENT_SRC})

	pybind11_add_module(smartredisPy
                        src/python/src/pysrobject.cpp
                        src/python/src/pylogcontext.cpp
	                    src/python/src/pyclient.cpp
                        src/python/src/pydataset.cpp
                        ${CLIENT_SRC}
                        src/python/bindings/bind.cpp)

	target_link_libraries(smartredisPy PUBLIC ${EXT_CLIENT_LIBRARIES})
	install(TARGETS smartredisPy
	        LIBRARY DESTINATION lib)
else()
	message("-- Skipping Python client build")
endif()
