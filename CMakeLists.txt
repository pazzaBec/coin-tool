cmake_minimum_required (VERSION 2.6)

project (COINPOOL)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")


###################################### Options ######################################
message("") # empty line
message("------------------------------- Options -------------------------------")

#
# Build Type: -DCMAKE_BUILD_TYPE=Debug|Release
# Default: Debug
#
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()
message("-- Build Type: ${CMAKE_BUILD_TYPE} (-DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})")
IF(CMAKE_BUILD_TYPE MATCHES Debug)
  set(CMAKE_CXX_FLAGS "-g -O0 -Wall")
  message("-- Debug build.")
ELSEIF(CMAKE_BUILD_TYPE MATCHES Release)
  set(CMAKE_CXX_FLAGS "-g -O2 -Wall")
  message("-- Release build.")
ELSE()
  set(CMAKE_CXX_FLAGS "-g -Wall")
  message("-- Some other build type.")
ENDIF()

list(APPEND CMAKE_CXX_FLAGS 
            "-DCHECK_PTHREAD_RETURN_VALUE"
            "-D_FILE_OFFSET_BITS=64"
            "-Wextra"
            "-Wsign-compare"
            "-Wunused-variable"
            "-Wno-unused-parameter"
            "-Woverloaded-virtual"
            "-Wpointer-arith"
            "-Wwrite-strings"
            "-march=native"
            "-rdynamic"
            " -pthread ")

list(LENGTH CMAKE_CXX_FLAGS CXX_FLAG_LEN)
message("-- CMAKE_CXX_FLAGS list length = " ${CXX_FLAG_LEN})
foreach(FLAG ${CMAKE_CXX_FLAGS})
message("--CMAKE_CXX_FLAGS: " ${FLAG})
endforeach

# compiler options
set(CMAKE_CXX_COMPILER "g++")
set(CMAKE_C_COMPILER "gcc")
set(CMAKE_CXX_COMPILER_ARG1 "-std=c++0x")

# The number of concurrent jobs when compiling a third-party library
if(JOBS)
  message("-- Concurrent Jobs: ${JOBS} (-DJOBS=${JOBS})")
  set(MAKE_JOBS "-j${JOBS}") 
else()
  message("-- Concurrent Jobs: 1 (-DJOBS=1)")
  set(MAKE_JOBS "") 
endif()

#
# Chain set
#
set(CHAIN_TYPE "UD")
set(CHAIN_SRC_ROOT /root/ulordtest)

#
# Chain Type
#
if(NOT CHAIN_TYPE)
  message(FATAL_ERROR "Chain type not defined! (-DCHAIN_TYPE=UD)")
endif()
# Ulord
if (CHAIN_TYPE STREQUAL "UD")
  set(CHAIN_NAME "ulord Core")
  set(POOL__DEB_PACKNAME_POSTFIX "${POOL__DEB_PACKNAME_POSTFIX}-ulord")
  set(POOL__DEFAULT_INSTALL_PREFIX "/work/udpool.ulord")

  add_definitions(-DCHAIN_TYPE_BTC)
  add_definitions(-DKAFKA_TOPIC_PREFIX="") # for compatibility with old deployments, let it empty.
  add_definitions(-DZOOKEEPER_NODE_POSTFIX="") # for compatibility with old deployments, let it empty.

# Unknown
else()
  message("Unknown chain type! (-DCHAIN_TYPE=${CHAIN_TYPE})")
  message(FATAL_ERROR "Supported chain type: BTC|BCH|UBTC|SBTC")
endif()

message("-- Chain Type: ${CHAIN_NAME} (-DCHAIN_TYPE=${CHAIN_TYPE})")

#
# Root Dir of Chain's Source Code
#
if(NOT CHAIN_SRC_ROOT)
  message(FATAL_ERROR "Root dir of chain's source code not defined! (-DCHAIN_SRC_ROOT=<path>)")
endif()
message("-- Root Dir of ${CHAIN_NAME}: ${CHAIN_SRC_ROOT} (-DCHAIN_SRC_ROOT=${CHAIN_SRC_ROOT})")
if (NOT EXISTS "${CHAIN_SRC_ROOT}/src/version.h")
  message(FATAL_ERROR "The source code of ${CHAIN_NAME} not exists!")
endif()


###################################### Libs ######################################
message("") # empty line
message("------------------------------- Libs -------------------------------")

#
# chain's libs
#
set(BITCOIN_LIBRARIES
  "${CHAIN_SRC_ROOT}/src/libbitcoin_common.a"
 # "${CHAIN_SRC_ROOT}/src/libbitcoinconsensus.la"
  "${CHAIN_SRC_ROOT}/src/libbitcoin_util.a"
  "${CHAIN_SRC_ROOT}/src/crypto/libbitcoin_crypto.a")

list(GET BITCOIN_LIBRARIES 0 BITCOIN_LIBRARIE)
if (NOT EXISTS ${BITCOIN_LIBRARIE})
  message(STATUS "building ${CHAIN_NAME}...")
  execute_process(WORKING_DIRECTORY "${CHAIN_SRC_ROOT}" COMMAND ./autogen.sh)
  execute_process(WORKING_DIRECTORY "${CHAIN_SRC_ROOT}" COMMAND ./configure --with-gui=no --disable-wallet --disable-tests --disable-bench)
  execute_process(WORKING_DIRECTORY "${CHAIN_SRC_ROOT}" COMMAND make ${MAKE_JOBS})
endif()

foreach(BITCOIN_LIBRARIE ${BITCOIN_LIBRARIES})
  if (NOT EXISTS ${BITCOIN_LIBRARIE})
    message(FATAL_ERROR "${BITCOIN_LIBRARIE} not exists!")
  endif()
endforeach()

find_package(LibPthread)
if(NOT PTHREAD_FOUND)
  message(FATAL_ERROR "libpthread not found!")
endif()

#
# libsecp256k1
#
set(secp256k1_DIR "${CHAIN_SRC_ROOT}/src/secp256k1")
set(secp256k1_LIBRARIES "${secp256k1_DIR}/.libs/libsecp256k1.a")
if (NOT EXISTS ${secp256k1_LIBRARIES})
  message(STATUS "build secp256k1...")
  execute_process(WORKING_DIRECTORY "${secp256k1_DIR}" COMMAND ./autogen.sh)
  execute_process(WORKING_DIRECTORY "${secp256k1_DIR}" COMMAND ./configure --enable-module-recovery)
  execute_process(WORKING_DIRECTORY "${secp256k1_DIR}" COMMAND make ${MAKE_JOBS})
endif()
if (NOT EXISTS ${secp256k1_LIBRARIES})
  message(FATAL_ERROR "build secp256k1 failed!")
endif()

find_package(OpenSSL)
if(NOT OPENSSL_FOUND)
  message(FATAL_ERROR "OpenSSL not found!")
endif()

#find_package(LibConfig)
#if(NOT LIBCONFIG_FOUND)
#  message(FATAL_ERROR "LibConfig not found!")
#endif()

#find_package(Glog)
#if(NOT GLOG_FOUND)
#message(FATAL_ERROR "Glog not found!")
#endif()

find_package(libzmq)
if(NOT LIBZMQ_FOUND)
  message(FATAL_ERROR "libzmq not found!")
endif()

#find_package(CURL)
#if(NOT CURL_FOUND)
#  message(FATAL_ERROR "libcurl not found!")
#endif()

find_package(LibEvent)
if(NOT LibEvent_FOUND)
  message(FATAL_ERROR "libevent2 not found!")
endif()

find_package(Boost 1.36.0 COMPONENTS thread filesystem system regex program_options)
if(NOT Boost_FOUND)
  message(FATAL_ERROR "Boost not found!")
endif()

#find_package(ZookeeperC)
#if(NOT ZOOKEEPER_FOUND)
#  message(FATAL_ERROR "libzookeeper-mt not found!")
#endif()

#find_package(LibGMP)
#if(NOT GMP_FOUND)
#  message(FATAL_ERROR "libgmp not found!")
#endif()

find_package(LibPthread)
if(NOT PTHREAD_FOUND)
  message(FATAL_ERROR "libpthread not found!")
endif()

#find_package(KafkaC)
#if(NOT KAFKA_FOUND)
#  message(FATAL_ERROR "librdkafka not found!")
#endif()
message("") # add an empty line

###################################### Targets ######################################
message("") # empty line
message("------------------------------- Targets -------------------------------")

include_directories(src ${CHAIN_SRC_ROOT}/src ${CHAIN_SRC_ROOT}/src/config ${CHAIN_SRC_ROOT}/src/secp256k1/include
                    ${OPENSSL_INCLUDE_DIR} ${Boost_INCLUDE_DIRS} ${LIBZMQ_INCLUDE_DIR} ${LIBEVENT_INCLUDE_DIR})
set(THIRD_LIBRARIES ${BITCOIN_LIBRARIES} ${OPENSSL_CRYPTO_LIBRARY} ${OPENSSL_SSL_LIBRARY} ${Boost_LIBRARIES}
                    ${LIBZMQ_LIBRARIES} ${LIBEVENT_LIB} ${LIBEVENT_PTHREADS_LIB} ${secp256k1_LIBRARIES} ${PTHREAD_LIBRARIES})

message("target::alert ******")
file(GLOB ALERT_SRC src/alert.cpp)
add_executable(alert ${ALERT_SRC})
target_link_libraries(alert ${THIRD_LIBRARIES})