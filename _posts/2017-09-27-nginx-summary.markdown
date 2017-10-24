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

###  nginx location 匹配规则


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
* "/index.html"不符合所有正则匹配的规则，所以采用规则B。
* "/documents/document.html",符合prefix string 里的规则B和规则C，但是根据最长匹配原则会匹配到C。
* "/images/1.gif" 符合B和D，因为前缀匹配的优先级更高，所以匹配到D。
* "/documents/1.jpg"符合B,C,E,因为E的优先级更高，所以会匹配到E。

### 实现restful风格路由
这块我先参考了一下php里面[yii](http://www.yiiframework.com/)的实现，yii里url mapping主要是在Request.php里的resolvePathInfo方法实现的

```java
protected function resolvePathInfo(){
  $pathInfo = $this->resolveRequestUri();
  // ...省略部分代码
  $scriptUrl = $this->getScriptUrl();
  $baseUrl = $this->getBaseUrl();
  if (strpos($pathInfo, $scriptUrl) === 0) {
    $pathInfo = substr($pathInfo, strlen($scriptUrl));
  } elseif ($baseUrl === '' || strpos($pathInfo, $baseUrl) === 0) {
    $pathInfo = substr($pathInfo, strlen($baseUrl));
  }
  // ...省略部分代码
  return (string) $pathInfo;
}

protected function resolveRequestUri(){
   // ...省略部分代码
  if (isset($_SERVER['REQUEST_URI'])) {
    $requestUri = $_SERVER['REQUEST_URI'];
    if ($requestUri !== '' && $requestUri[0] !== '/') {
      $requestUri = preg_replace('/^(http|https):\/\/[^\/]+/i', '', $requestUri);
    }
  }   
  // ...省略部分代码
  return $requestUri;
}

public function getBaseUrl(){
  if ($this->_baseUrl === null) {
    $this->_baseUrl = rtrim(dirname($this->getScriptUrl()), '\\/');
  }
  return $this->_baseUrl;
}

public function getScriptUrl(){
  if ($this->_scriptUrl === null) {
    $scriptFile = $_SERVER['SCRIPT_FILENAME'];
    $scriptName = basename($scriptFile);
    if (basename($_SERVER['SCRIPT_NAME']) === $scriptName){
      $this->_scriptUrl = $_SERVER['SCRIPT_NAME'];
    }
     // ...省略部分代码
  }

  return $this->_scriptUrl;
}
```

省略的一些不关键代码，可以看出主要是根据php内置的 $_SERVER['SCRIPT_NAME']，$_SERVER['REQUEST_URI']，$_SERVER['SCRIPT_FILENAME'] 来计算pathinfo的，逻辑不在赘述。在nginx环境里，php的内置变量主要是由[ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html) 和[ngx_http_fastcgi_module](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)模块提供。

在这里我们先取到$SERVER['REQUEST_URI']，也就是原始的请求url。

```
$request_uri
full original request URI (with arguments)
```

然后我们要从原始的uri里提取我们所要的pathinfo，我们知道yii支持如
/xxx/index.php/controller/action 以及/xxx/controller/action 的restful风格的url的，而我们所需要的pathinfo实际上是这个controller/action的部分，所以在这里我们要借助$_SERVER['SCRIPT_FILENAME']，$_SERVER['SCRIPT_NAME']这两个内置变量。

$_SERVER['SCRIPT_NAME'] 对应了 $fastcgi_script_name

$_SERVER['SCRIPT_FILENAME']对应了 fastcgi_param SCRIPT_FILENAME

```
$fastcgi_script_name
request URI or, if a URI ends with a slash, request URI with an index file name configured by the fastcgi_index directive appended to it. This variable can be used to set the SCRIPT_FILENAME and PATH_TRANSLATED parameters that determine the script name in PHP. For example, for the “/info/” request with the following directives
fastcgi_index index.php;
fastcgi_param SCRIPT_FILENAME /home/www/scripts/php$fastcgi_script_name;
the SCRIPT_FILENAME parameter will be equal to “/home/www/scripts/php/info/index.php”.
When using the fastcgi_split_path_info directive, the $fastcgi_script_name variable equals the value of the first capture set by the directive.
```

需要注意的是这里的request URI不是origin request URI，internal redirect的也是包含在内的，实际上你用restful like url，脚本的入口肯定是固定的一个，所以肯定会用try_files，或者是redirect这样的指令的。我们在本地开发的时候一般都会多个php项目，然后一个nginx容器，依靠不同的base url来做区分，这样的话，在nginx里我们可以把document_root指向项目的父目录，然后用try_files，去把请求重定向到我们的对应的入口上如：

```nginx
	location /sanguo2{
	    try_files $uri $uri/ /sanguo2/index.php?$args;
	}
```

这时的$fastcgi_script_name  就是/sanguo2/index.php，而SCRIPT_FILENAME一般我们会设置为$document_root$fastcgi_script_name，这样才能找到这样的文件，这是yii里的baseurl 会取dirname($_SERVER['SCRIPT_NAME'] )，会得到/sanguo2，如果我们请求/sanguo2/home/index，那么yii会得到$_SERVER['REQUEST_URI'] = /sanguo2/home/index，baseurl=/sanguo2，截取以后会得到最后的pathinfo=home/index

### openresty的实现

由于openresty跟fastcgi并不是一个层面的东西，所以显然我们没法使用ngx_http_fastcgi_module的变量了，但是还记得我们之前说的如果是restful风格的url的那么脚本的入口显然是固定的，所以肯定会有internal redirect，所以我们做如下配置：

```nginx
location /test {
	try_files $uri $uri/ /test/index.lua?$args;
}

location ~ /(.*\.lua$) {
	content_by_lua_file $1;
}
```

访问http://127.0.0.1:8081/test/main/dashboard 时，会得到输出

```
hello i am index.lua
ngx.var.uri = /test/index.lua
ngx.req.get_method() = GET
ngx.var.request_uri = /test/main/dashboard
ngx.var.request_filename = /Users/xinlei.fan/Documents/workspace/June/html/test/index.lua
```

可以看出ngx.var.uri，由于发生了internal redirect，所以值是跳转后的uri，我们就可以对这个uri取dirname来得到baseurl。然后ngx.var.request_uri是原始请求的uri，我们从这里trim掉baseurl来得到最后的pathinfo。由于我们的跳转写在try_files的最后，所以当访问test下的具体文件甚至是lua时都能得到正确的结果。
