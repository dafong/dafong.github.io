---
layout:     post
title:      "不要使用ngx.ctx"
subtitle:   ""
date:       2017-10-24
author:     "Dafong"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - openresty
---

openresty 由于在处理一个请求的时候是使用一个协程来处理，所以一个woker进程在同一时间内是有可能处理
多个请求的。在实现一个简单的web框架的时候，我们避免不了要将一些数据结构绑定到一个请求的生命周期上，那
么如何做呢？openresty官方的文档上提供了以下几种做法：

## 方法

### 使用ngx.var
使用ngx.var有很大的局限，比如说这东西不能create on-the-fly，只能先在nginx的配置文件里声明一下
```
set $some_var "";
```
另外一个支持的数据类型也只用string和number，这样显然不行。

### 使用ngx.ctx
ngx.ctx 跟ngx.var一样，声明周期都是跟当前的请求绑定的，可以支持任意类型的数据，但代价相对昂贵。


### 手动传参
简单来说就是把请求所共享的数据结构都封装到一个对象里来，然后在使用到的地方(函数调用)，当做参数传进去。

## 验证

* ngx.var
这个局限性太大了，所以不在考虑之列

* ngx.ctx
就是在一个请求内，从ctx里存取100万次数据。代码如下

```lua
ngx.ctx.t = {cnt = 0}
for i=1,1000000,1 do
    ngx.ctx.t.cnt = ngx.ctx.t.cnt + 1
end
ngx.say(ngx.ctx.t.cnt)

```

用ab压了一下，慢到怀疑人生啊
Requests per second:    1.16 [#/sec] (mean)

* 手动传参
作为对比，代码如下

```lua
local t = {cnt = 0}
for i=1,1000000,1 do
    t.cnt = t.cnt + 1
end
ngx.say(t.cnt)
```
还有使用元表的

```lua
local t = {}
local m = {cnt = 0}
setmetatable(t,{__index = function(t,k)
    return m[k]
end})
for i=1,1000000,1 do
    t.cnt = t.cnt + 1
end
ngx.say(t.cnt)

```
都非常快。

在我的老mac air上跑出
Requests per second:    586.42 [#/sec] (mean)
