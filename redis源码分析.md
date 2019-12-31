

# main 启动流程

```mermaid
sequenceDiagram
    redis.c ->> redis.h.c: main
    redis.c ->> locale.h: setlocale
    redis.c ->> zmalloc.c: zmalloc_enable_thread_safeness
    redis.c ->> zmalloc.c: zmalloc_set_oom_handler
    redis.c ->> stdlib.h: srand
    redis.c ->> time.c: gettimeofday
    redis.c ->> dict.c: dictSetHashFunctionSeed
    redis.c ->> redis.c: checkForSentinelMode
    redis.c ->> redis.c: initServerConfig
    redis.c ->> redis.c: getRandomHexChars(set runid)
    redis.c ->> config.c: resetServerSaveParams
    redis.c ->> config.c: appendServerSaveParams
    opt only sentinel_mode 
    redis.c ->> sentinel.c: initSentinelConfig 
    redis.c ->> sentinel.c: initSentinel 
    end
    redis.c ->> config.c: loadServerConfig
    config.c ->> stdio.h: fopen
    config.c ->> stdio.h: fgets
    config.c ->> config.c: loadServerConfigFromString
    config.c -->> redis.c: loadServerConfig
    opt only daemonize
    redis.c ->> redis.c: daemonize
    end
    redis.c ->> redis.c: initServer
    redis.c ->> redis.c: setupSignalHandlers
    redis.c ->> redis.c: createSharedObjects
    redis.c ->> redis.c: adjustOpenFilesLimit
    redis.c ->> ae.c: aeCreateEventLoop
    ae.c ->> ae.c: aeApiCreate
    ae.c ->> epoll.h: epoll_create
    ae.c -->> redis.c: aeCreateEventLoop
    redis.c ->> redis.c: listenToPort
    redis.c ->> anet.c: anetTcpServer
    anet.c ->> anet.c: _anetTcpServer
    anet.c ->> netdb.h: getaddrinfo
    anet.c ->> socket.h: socket
    anet.c ->> anet.c: anetSetReuseAddr
    anet.c ->> anet.c: anetListen
    anet.c ->> socket.h: bind
    anet.c ->> socket.h: listen
    anet.c ->> netdb.h: freeaddrinfo
    anet.c -->> redis.c: anetTcpServer
    redis.c ->> anet.c: anetNonBlock
    anet.c ->> anet.c: anetSetBlock
    anet.c -->> redis.c: anetNonBlock
    redis.c ->> unistd.h: unlink
    redis.c ->> anet.c: anetUnixServer
    anet.c ->> anet.c: anetCreateSocket
    anet.c ->> socket.h: socket
    anet.c ->> anet.c: anetSetReuseAddr
    anet.c ->> socket.h: setsockopt
    anet.c ->> anet.c: anetListen
    anet.c ->> socket.h: bind
    anet.c ->> socket.h: listen
    anet.c -->> redis.c: anetUnixServer
    redis.c ->> anet.c: anetNonBlock
    anet.c ->> anet.c: anetSetBlock
    anet.c -->> redis.c: anetNonBlock
    redis.c ->> redis.c: evictionPoolAlloc
    redis.c ->> aof.c: aofRewriteBufferReset
    redis.c ->> redis.c: resetServerStats
    redis.c ->> redis.c: updateCachedTime
    opt install serverCron
        redis.c ->> ae.c: aeCreateTimeEvent
        ae.c ->> ae.c: aeAddMillisecondsToNow
        ae.c -->> redis.c: aeCreateTimeEvent
    end
    opt install acceptTcpHandler
        redis.c ->> ae.c: aeCreateFileEvent
        ae.c ->> ae_epoll.c: aeApiAddEvent
        ae.c ->> epoll.h: epoll_ctl
        ae_epoll.c -->> ae.c: aeApiAddEvent
        ae.c -->> redis.c: aeCreateFileEvent
    end
    opt install acceptUnixHandler
        redis.c ->> ae.c: aeCreateFileEvent
        ae.c ->> ae_epoll.c: aeApiAddEvent
        ae.c ->> epoll.h: epoll_ctl
        ae_epoll.c -->> ae.c: aeApiAddEvent
        ae.c -->> redis.c: aeCreateFileEvent
    end
    redis.c ->> fcntl2.h: open(AOF file)
    opt only cluster_enabled
        redis.c ->> cluster.c: clusterInit
        cluster.c ->> cluster.c: clusterCloseAllSlots
        cluster.c ->> cluster.c: clusterLockConfig
        cluster.c ->> cluster.c: clusterLoadConfig
        cluster.c ->> cluster.c: createClusterNode
        cluster.c ->> cluster.c: clusterAddNode
        cluster.c ->> cluster.c: clusterSaveConfigOrDie
        cluster.c ->> cluster.c: clusterSaveConfig
        cluster.c ->> redis.c: listenToPort
        redis.c -->> cluster.c: listenToPort

        opt install clusterAcceptHandler
            redis.c ->> ae.c: aeCreateFileEvent
            ae.c ->> ae_epoll.c: aeApiAddEvent
            ae.c ->> epoll.h: epoll_ctl
            ae_epoll.c -->> ae.c: aeApiAddEvent
            ae.c -->> redis.c: aeCreateFileEvent
        end
        cluster.c ->> cluster.c: resetManualFailover
        cluster.c -->> redis.c: clusterInit
    end
    redis.c ->> replication.c: replicationScriptCacheInit
    redis.c ->> scripting.c: scriptingInit
    scripting.c ->> lua.h: lua_open
    scripting.c ->> scripting.c: luaLoadLibraries
    scripting.c ->> scripting.c: luaRemoveUnsupportedFunctions
    scripting.c ->> lua.h: lua_newtable
    scripting.c ->> lapi.c: lua_pushstring
    scripting.c ->> lua.h: lua_pushcfunction
    scripting.c ->> lapi.c: lua_settable
    scripting.c ->> lua.h: lua_setglobal
    scripting.c ->> scripting.c: scriptingEnableGlobalsProtection
    scripting.c -->> redis.c: scriptingInit
    redis.c ->> showlog.c: slowlogInit
    redis.c ->> latency.c: latencyMonitorInit
    redis.c ->> bio.c: bioInit
    redis.c -->> redis.c: initServer
    opt only daemonize
        redis.c -->> redis.c: createPidFile
    end
    redis.c ->> redis.c: redisSetProcTitle
    redis.c ->> redis.c: redisAsciiArt
    redis.c ->> redis.c: checkTcpBacklogSettings
    opt if !sentinel_mode
        redis.c ->> redis.c: linuxMemoryWarnings
        redis.c ->> redis.c: loadDataFromDisk
        opt only AOF
            redis.c ->> aof.c: loadAppendOnlyFile
            aof.c ->> stdio.h: fopen
            aof.c ->> aof.c: createFakeClient
            aof.c ->> rdb.c: startLoading
            opt while
                aof.c ->> rdb.c: loadingProgress
                aof.c ->> networking.c: processEventsWhileBlocked
                aof.c ->> redis.c: lookupCommand
                redis.c ->> redisCommand: proc
            end
            aof.c ->> stdio.h: fclose
            aof.c ->> aof.c: freeFakeClient
            aof.c ->> rdb.c: stopLoading
            aof.c ->> aof.c: aofUpdateCurrentSize
            aof.c ->> latency.h: latencyStartMonitor
            aof.c ->> config.c: redis_fstat
            aof.c ->> latency.h: latencyEndMonitor
            aof.c ->> latency.h: latencyAddSampleIfNeeded
            aof.c -->> redis.c: loadAppendOnlyFile
        end
        opt only not AOF
            redis.c ->> rdb.c: rdbLoad
            rdb.c ->> stdio.h: fopen
            rdb.c ->> rio.c: rioInitWithFile
            rdb.c ->> rio.c: rioRead
            rdb.c ->> stdlib.c: atoi
            rdb.c ->> rdb.c: startLoading
            rdb.c ->> stdio.h: fclose
            rdb.c ->> rdb.c: stopLoading
            rdb.c -->> redis.c: rdbLoad
        end
        opt only cluster_enabled
        redis.c ->> cluster.c: verifyClusterConfigWithData
        cluster.c -->> redis.c: verifyClusterConfigWithData
        end
    end
    opt if sentinel_mode
        redis.c ->> sentinel.c: sentinelIsRunning
    end
    redis.c ->> ae.c: aeSetBeforeSleepProc
    ae.c -->> redis.c: aeSetBeforeSleepProc
    redis.c ->> ae.c: aeMain
    opt while !stop
        ae.c ->> ae.c: aeProcessEvents
    end
    ae.c -->> redis.c: aeMain
    redis.c ->> ae.c: aeDeleteEventLoop
    ae.c-> ae.c: aeApiFree
    ae.c -->> redis.c: aeDeleteEventLoop
```



# 命令处理流程
```mermaid
sequenceDiagram
    opt install acceptTcpHandler
        redis.c ->> ae.c: aeCreateFileEvent
        ae.c ->> ae_epoll.c: aeApiAddEvent
        ae.c ->> epoll.h: epoll_ctl
        ae_epoll.c -->> ae.c: aeApiAddEvent
        ae.c -->> redis.c: aeCreateFileEvent
    end

    opt 执行 acceptTcpHandler
        networking.c ->> networking.c: acceptTcpHandler
        networking.c ->> anet.c: anetTcpAccept
        anet.c ->> anet.c: anetGenericAccept
        anet.c -->> networking.c: anetTcpAccept
        networking.c ->> networking.c: acceptCommonHandler
        networking.c ->> networking.c: createClient
        networking.c ->> anet.c: anetNonBlock
        anet.c ->> anet.c: anetSetBlock
        anet.c -->> networking.c: anetNonBlock
        networking.c ->> anet.c: anetEnableTcpNoDelay
        anet.c ->> anet.c: anetSetTcpNoDelay
        anet.c -->> networking.c: anetEnableTcpNoDelay
        opt install readQueryFromClient
            networking.c ->> ae.c: aeCreateFileEvent
            ae.c ->> ae_epoll.c: aeApiAddEvent
            ae.c ->> epoll.h: epoll_ctl
            ae_epoll.c -->> ae.c: aeApiAddEvent
            ae.c -->> networking.c: aeCreateFileEvent
        end
    end

    opt 执行 readQueryFromClient
        networking.c ->> unistd.c: read
        unistd.c -->> networking.c: read
        networking.c ->> networking.c: processInputBuffer
        networking.c ->> redis.c: processCommand
        redis.c ->> redis.c: lookupCommand
        opt MULTI commands queue
            redis.c ->> multi.c: queueMultiCommand
            multi.c -->> redis.c: queueMultiCommand
            redis.c ->> networking.c: addReply
            networking.c ->> networking.c: prepareClientToWrite
            opt install sendReplyToClient
                networking.c ->> ae.c: aeCreateFileEvent
                ae.c ->> ae_epoll.c: aeApiAddEvent
                ae.c ->> epoll.h: epoll_ctl
                ae_epoll.c -->> ae.c: aeApiAddEvent
                ae.c -->> networking.c: aeCreateFileEvent
            end
            networking.c ->> networking.c: _addReplyToBuffer
            networking.c -->> redis.c: addReply
        end
        opt Exec
            redis.c ->> redis.c: call
            
            redis.c ->> redisCommand: proc
            redisCommand ->> networking.c: addReply
            networking.c ->> networking.c: prepareClientToWrite
            opt install sendReplyToClient
                networking.c ->> ae.c: aeCreateFileEvent
                ae.c ->> ae_epoll.c: aeApiAddEvent
                ae.c ->> epoll.h: epoll_ctl
                ae_epoll.c -->> ae.c: aeApiAddEvent
                ae.c -->> networking.c: aeCreateFileEvent
            end
            networking.c ->> networking.c: _addReplyToBuffer
            networking.c -->> redisCommand: addReply
            redisCommand -->> redis.c: proc
        end
        redis.c -->> networking.c: processCommand
        networking.c ->> networking.c: resetClient
    end

    opt 执行 sendReplyToClient
        networking.c ->> unistd.c: write
        unistd.c -->> networking.c: write
        networking.c ->> ae.c: aeDeleteFileEvent
        ae.c ->> ae_epoll.c: aeApiDelEvent
        ae_epoll.c -->> ae.c: aeApiDelEvent
        ae.c -->> networking.c: aeDeleteFileEvent
    end
```

# 命令处理流程详情
1. redis-server 启动
    * **main 函数**中创建 socket 监听 （redis.c#listenToPort -> ipfd
    * **main 函数**中添加 accept 事件 (ae.c#aeCreateFileEvent -> acceptTcpHandler, ipfd
    * **main 函数**中循环处理事件, 等待 socket 连接 （ae.c#aeMain
2. redis-server 接收命令
    * 客户端 socket 连接到 redis-server
    * **acceptTcpHandler 函数**中接收, 并创建客户端实体
    * 添加 read 事件 (ae.c#aeCreateFileEvent -> readQueryFromClient, redisClient.fd
    * 客户端 socket 传送命令到 redis-server
    * **readQueryFromClient 函数**中读命令(redisClient.querybuf), 解析命令, 执行命令, 写结果(redisClient.buf)
    * 添加 write 事件 (ae.c#aeCreateFileEvent -> sendReplyToClient, redisClient.fd
3. redis-server 返回结果
    * **sendReplyToClient 函数**将命令执行结果发送给客户端
    * 移除 write 事件 (ae.c#aeDeleteFileEvent -> redisClient.fd