---
title: 在M1上使用CMake编译redis6.2.14
slug: m1-cmake-compile-redis6
desc: 本文介绍了如何在 macOS M1 系统上使用 CMake 编译 Redis 6.2.14 版本的详细步骤。由于 Redis 官方在 6.2.14 版本之前对 ARM 架构的支持存在问题，因此推荐使用 6.2.14 或更高版本进行编译。文章提供了从创建 CMakeLists.txt 文件到编译和安装 Redis 的完整流程，包括各个依赖库（如 hdr_histogram、hiredis、linenoise、lua）的 CMake 配置。通过本文的指南，读者可以在 macOS M1 系统上顺利编译并安装 Redis 6.2.14。
featuredImage: https://sonder.vitah.me/featured/833a09cb7a72909c6052e66a0fb2fafe.webp
categories:
  - Database
tags:
  - Redis
---

在 MAC M1 系统上编译 redis 源码会有如下问题： [https://github.com/redis/redis/issues/12585](https://github.com/redis/redis/issues/12585)
所以我们需要下载 6.2.14 及以后的版本。

## 相关版本说明

- **redis：6.2.14**
- **macOS：14.4.1**
- **CMake：3.19.6**

## 对应的 CMakeLists. txt 文件

使用 `cmake` 编译 `redis` 源码的话需要创建对应的 `CMakeLists.txt` 文件，根据下述步骤创建对应的 CMakeLists. txt 文件。

### /deps/hde_histogram/CMakeLists. txt

```c
add_library(hdr_histogram hdr_histogram.c)
```

### /deps/linenoise/CMakeLists. txt

```c
add_library(linenoise linenoise.c)
```

### /deps/lua/CMakeLists. txt

```c
add_subdirectory(hdr_histogram)
add_subdirectory(hiredis)
add_subdirectory(linenoise)
add_subdirectory(lua)
```

### /deps/CMakeLists. txt

```c
add_subdirectory(hdr_histogram)
add_subdirectory(hiredis)
add_subdirectory(linenoise)
add_subdirectory(lua)
```

### CMakeLists. txt

```c
# cmake最低版本要求
cmake_minimum_required(VERSION 3.13)

#项目信息
project(redis6.2.14_debug)

#debug模式编译
set(CMAKE_BUILD_TYPE Debug CACHE STRING "set build type to debug")

# GCC编译选项 O2 第二级别优化
## -Wall：开启警告提示
# -std=c99：C99标准
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O2")
# -std=c++11, -std=c99
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -pedantic -DREDIS_STATIC=''")
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -W -Wno-missing-field-initializers")

#可执行文件的输出目录
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/)

#安装路径
set(CMAKE_INSTALL_PREFIX ${PROJECT_SOURCE_DIR}/target/)

#依赖库目录
#Redis源码编译需要依赖的自定义库的路径，在主目录deps下
set(DEPS_PATH ${CMAKE_CURRENT_SOURCE_DIR}/deps)

#依赖共享库, 根据原Rerdis中的Makefile，编译源码需要依赖的系统共享库
set(SHARED_LIBS -lm -ldl -lpthread)

#在编译Redis之前需要执行src目录下的 mkreleasehdr.sh 脚本生成 release.h
execute_process(COMMAND sh ${CMAKE_CURRENT_SOURCE_DIR}/src/mkreleasehdr.sh WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src)

#include directories
include_directories(
        ${DEPS_PATH}/hdr_histogram
        ${DEPS_PATH}/hiredis
        ${DEPS_PATH}/lua/src
        ${DEPS_PATH}/linenoise
)

# 添加需要编译子目录
add_subdirectory(deps)

link_directories(${DEPS_PATH}/hdr_histogram)
link_directories(${DEPS_PATH}/hiredis)
link_directories(${DEPS_PATH}/lua/src)
link_directories(${DEPS_PATH}/linenoise)

# redis-server 来源src/Makefile的REDIS_SERVER_OBJ 修改.o为.c
set(REDIS_SERVER_LIST src/adlist.c src/quicklist.c src/ae.c src/anet.c src/dict.c src/server.c src/sds.c src/zmalloc.c src/lzf_c.c src/lzf_d.c src/pqsort.c src/zipmap.c src/sha1.c src/ziplist.c src/release.c src/networking.c src/util.c src/object.c src/db.c src/replication.c src/rdb.c src/t_string.c src/t_list.c src/t_set.c src/t_zset.c src/t_hash.c src/config.c src/aof.c src/pubsub.c src/multi.c src/debug.c src/sort.c src/intset.c src/syncio.c src/cluster.c src/crc16.c src/endianconv.c src/slowlog.c src/scripting.c src/bio.c src/rio.c src/rand.c src/memtest.c src/crcspeed.c src/crc64.c src/bitops.c src/sentinel.c src/notify.c src/setproctitle.c src/blocked.c src/hyperloglog.c src/latency.c src/sparkline.c src/redis-check-rdb.c src/redis-check-aof.c src/geo.c src/lazyfree.c src/module.c src/evict.c src/expire.c src/geohash.c src/geohash_helper.c src/childinfo.c src/defrag.c src/siphash.c src/rax.c src/t_stream.c src/listpack.c src/localtime.c src/lolwut.c src/lolwut5.c src/lolwut6.c src/acl.c src/gopher.c src/tracking.c src/connection.c src/tls.c src/sha256.c src/timeout.c src/setcpuaffinity.c src/monotonic.c src/mt19937-64.c)

#redis-cli
set(REDIS_CLI_LIST src/anet.c src/adlist.c src/dict.c src/redis-cli.c src/zmalloc.c src/release.c src/ae.c src/crcspeed.c src/crc64.c src/siphash.c src/crc16.c src/monotonic.c src/cli_common.c src/mt19937-64.c)

# redis-benchmark
set(REDIS_BENCHMARK_LIST src/ae.c src/anet.c src/redis-benchmark.c src/adlist.c src/dict.c src/zmalloc.c src/release.c src/crcspeed.c src/crc64.c src/siphash.c src/crc16.c src/monotonic.c src/cli_common.c src/mt19937-64.c)

#生成可执行文件
add_executable(redis-server ${REDIS_SERVER_LIST})
add_executable(redis-benchmark ${REDIS_BENCHMARK_LIST})
add_executable(redis-cli ${REDIS_CLI_LIST})

# 将目标文件与库文件进行链接
target_link_libraries(redis-server hdr_histogram hiredis lua ${SHARED_LIBS})
target_link_libraries(redis-benchmark hdr_histogram hiredis ${SHARED_LIBS})
target_link_libraries(redis-cli hdr_histogram linenoise hiredis ${SHARED_LIBS})

#add_custom_command不直接执行，target依赖成立后执行
ADD_CUSTOM_COMMAND(
        TARGET redis-server
        COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/redis-server  ${CMAKE_CURRENT_SOURCE_DIR}/redis-sentinel
        COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/redis-server  ${CMAKE_CURRENT_SOURCE_DIR}/redis-check-rdb
        COMMAND cp ${CMAKE_CURRENT_SOURCE_DIR}/redis-server  ${CMAKE_CURRENT_SOURCE_DIR}/redis-check-aof
)

#执行make install执行以下安装程序
install(TARGETS redis-server DESTINATION bin)
install(TARGETS redis-benchmark DESTINATION bin)
install(TARGETS redis-cli DESTINATION bin)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/redis-server DESTINATION bin RENAME redis-check-rdb)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/redis-server DESTINATION bin RENAME redis-check-aof)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/redis-server DESTINATION bin RENAME redis-sentinel)
```
