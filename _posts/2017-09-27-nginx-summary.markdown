---
layout:     post
title:      "Nginx小记"
subtitle:   ""
date:       2017-09-27
author:     "Dafong"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - openresty
---

研究[openresty](https://openresty.org)的时候准备自己写一个简单的web framework,以前对nginx的location匹配规则,以及如何在脚本这一端实现一套restful风格的url路由都没有细研究过，这次也算是做个记录。

### nginx location 匹配规则

|     sytax     |          define           |
| :-----------: | :-----------------------: |
|       =       |           精确匹配            |
| prefix string |   nginx最优先检查,最长匹配的被作为候选   |
|      ^~       | 前缀匹配，如果存在最长的匹配则不检查后面的正则匹配 |
|       ~       |        大小写敏感的正则匹配         |
|      ~*       |        大小写不敏感的正则匹配        |

所以总结起来的优先级就是

```
= > ^~ > ~ = ~* > prefix string
```

同为正则匹配的~和~*，匹配的优先级按照在配置文档中出现的顺序决定。

```nginx
location = / {
    [ configuration A ]
}

location / {
    [ configuration B ]
}

location /documents/ {
    [ configuration C ]
}

location ^~ /images/ {
    [ configuration D ]
}

location ~* \.(gif|jpg|jpeg)$ {
    [ configuration E ]
}
```

* "/" 会被精确匹配到A。
* "/index.html"不符合所有正则匹配的规则，所以采用规则C。
* "/documents/document.html",符合prefix string 里的规则B和规则C，但是根据最长匹配原则会匹配到C。
* "/images/1.gif" 符合B和D，因为前缀匹配的优先级更高，所以匹配到D。
* "/documents/1.jpg"符合B,C,E,因为E的优先级更高，所以会匹配到E。

### 