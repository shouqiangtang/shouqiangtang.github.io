---
layout: post
title:  Redis分布式锁Bug修复
date:   2020-06-12 11:45:00 +0800
categories: openresty
tags: redis setnx
---

# 问题现象：
同事反映客户端无法和服务器建立websocket连接，一直尝试重连但是都被服务器拒绝。服务器正常运行很久了，但是没有碰到过这种问题，另外听同事说今天路由器断电重启过。

# nginx日志如下：

```
2020/06/12 09:23:41 [info] 69177#69177: *29781 send() failed (32: Broken pipe), client: 127.0.0.1, server: ngx.******, request: "GET /v2/*******/B10902/?token=24181****** HTTP/1.1", host: "127.0.0.1:5111"
2020/06/12 09:23:41 [error] 69177#69177: *29781 [lua] ndscloud.lua:3653: failed to send frame: failed to send frame: broken pipe, client: 127.0.0.1, server: ngx.******, request: "GET /v2/******/B10902/?token=24181****** HTTP/1.1", host: "127.0.0.1:5111"
2020/06/12 09:23:41 [info] 69177#69177: *29781 [lua] ndscloud.lua:4280: [cdk_R-BC60AE1E-******] Kill thread connOnlyOne, client: 127.0.0.1, server: ngx.******, request: "GET /v2/******/B10902/?token=24181****** HTTP/1.1", host: "127.0.0.1:5111"
2020/06/12 09:23:41 [error] 69177#69177: *29781 [lua] ndscloud.lua:4292: [cdk_R-BC60AE1E-******] failed to send the close frame: fatal error already happened, client: 127.0.0.1, server: ngx.******, request: "GET /v2/******/B10902/?token=24181****** HTTP/1.1", host: "127.0.0.1:5111"
2020/06/12 09:23:43 [info] 69178#69178: *29790 [lua] ndscloud.lua:3691: [cdk_R-BC60AE1E-******] receive frame: {"did":"cdk_R-BC60AE1E-******","nm":"cdk_R-BC60AE1E-******","dt":"1","act":"1"} text nil, client: 127.0.0.1, server: ngx.******, request: "GET /v2/******/B10902/?token=24181****** HTTP/1.1", host: "127.0.0.1:5111"
2020/06/12 09:23:43 [error] 69178#69178: *29790 [lua] ndscloud.lua:477: [cdk_R-BC60AE1E-******] force terminate, because multi connections at the same time, client: 127.0.0.1, server: ngx.******, request: "GET /v2/******/B10902/?token=24181****** HTTP/1.1", host: "127.0.0.1:5111"
2020/06/12 09:23:43 [info] 69178#69178: *29790 [lua] ndscloud.lua:4280: [cdk_R-BC60AE1E-******] Kill thread connOnlyOne, client: 127.0.0.1, server: ngx.*, request: "GET /v2/******/B10902/?token=24181****** HTTP/1.1", host: "127.0.0.1:5111"
```

# 问题代码：

```
-- 限制同一用户/设备登录云中控，同一时刻只能接受一个连接
function NdsCloud:connOnlyOne()
    local red = redislib:cloud()

    -- 确保同一时刻多个客户端连接中控时，只能接入一个
    -- 设置互斥锁
    local existed, err = red:setnx(self.connLockKey, 1)
    if not existed then
        ngx.log(ngx.ERR, "[", self.id, "] failed to setnx ", self.connLockKey)
        return self:forceTerminate(red)
    end
    -- 如果锁已存在则断开Websocket连接
    if tonumber(existed) == 0 then
        ngx.log(ngx.ERR, "[", self.id, "] force terminate, because multi connections at the same time");
        return self:forceTerminate(red)
    end
    -- 设置互斥锁过期时间为5s
    local res, err = red:expire(self.connLockKey, 5)
    if not res then
        ngx.log(ngx.ERR, "[", self.id, "] failed to expire ", self.connLockKey, " 5")
        return self:terminate(red)
    end
    
    ......
end


-- 主函数
function NdsCloud:run(raw_unit_id)
    -- 限制同一用户/设备登录云中控，同一时刻只能接受一个连接
    self.spawnthreads["authrize"] = ngx.thread.spawn(self.connOnlyOne, self)

    ....省略....

    -- main loop : 接受websocket消息
    while true do
        local data, typ, err = self.wb:recv_frame()
        if not data then
            if not string.find(err, "timeout", 1, true) then
                local bytes, err = self.wb:send_ping()
                if not bytes then
                    ngx.log(ngx.ERR, "[", self.id, "] failed to receive a frame: ", err)
                    return self.isregistered and self:terminateoffline() or self:terminate()
                end
            end
        end

        if typ == "close" then
            ngx.log(ngx.ERR, "[", self.id, "] receive close frame from client")
            return self.isregistered and self:terminateoffline() or self:terminate()
        end

        if typ == "ping" then
            local bytes, err = self.wb:send_pong('pong')
            if not bytes then
                ngx.log(ngx.ERR, "[", self.id, "] failed to send frame: ", err)
                return self.isregistered and self:terminateoffline() or self:terminate()
            end
        elseif typ == "pong" then
            self:record_heartbeat()
        elseif typ == "text" then
            if tostring(raw_unit_id) ~= tostring(self.unitid) then
                return self:terminateoffline(nil)
            end
            if self.debug == 1 then
                ngx.log(ngx.INFO, "[", self.id, "] receive frame: ", data, " ", typ, " ", err)
            end
            self:scheduler(data)
            self:record_heartbeat()
        elseif typ == "binary" then
            -- ngx.log(ngx.INFO, "[", self.id, "] receive frame: ", string.len(data), " ", binary:hexEncode(data), " ", typ, " ", err)
            self:scheduler_binary(data)
            self:record_heartbeat()
        end
    end
end
```


# 问题分析：

从日志可以看到服务端一直报"force terminate, because multi connections at the same time"，看代码可直是由于self.connLockKey一直存在造成的（正常情况下self.connLockKey会5s后过期或注册成功后会被删除掉）。然后是什么原因造成的self.connLockKey一直存在呢？从日志上可以看出由于websocket发送消息失败时，服务端会kill掉connOnlyOne协程，这样的话可能无法设置self.connLockKey的过期时间，所以造成客户端无法和服务器建立连接。


# 解决方案：

使用set nc:connection:lock:B8467:ws-25 1 NX EX 5来实现原子操作，setnx和expire操作合并成一条指令。改进代码如下：

```
function NdsCloud:connOnlyOne()
    local red = redislib:cloud()

    -- 确保同一时刻多个客户端连接中控时，只能接入一个
    -- 设置互斥锁
    local ok, err = red:set(self.connLockKey, 1, "NX", "EX", 5)
    if not ok then
        ngx.log(ngx.ERR, "[", self.id, "] failed to set ", self.connLockKey)
        return self:forceTerminate(red)
    end
    ngx.log(ngx.ERR, "[", self.id, "] set ", self.connLockKey, " 1 NX EX 5")
    -- 如果锁已存在则断开Websocket连接
    if tostring(ok) ~= "OK" then
        ngx.log(ngx.ERR, "[", self.id, "] force terminate, because multi connections at the same time");
        return self:forceTerminate(red)
    end
    
    .......
end
```

