---
title: JSONP原理解析
date: '2017-12-03T20:21:03.284Z'
---

## 前言
我工作以来接触的第一个项目就是前后端分离的，前端静态文件有自己独立域名，通过接口来获取数据进行渲染等操作。  
跨域的方法不需要多言，随便一搜，就有很多，但最常用不外乎jsonp和CORS。jsonp着重于前端，也算是前端Hack技巧，CORS重于后端，服务端需要配置的地方会较多。  
这篇解析一下jsonp的实现原理。
## 基本原理
基本原理很容易说明白，在html页面中有一些标签是不受跨域限制的，比如img，script，link等。如果把我们需要的数据，放在一个js文件里面，这时，我们就能突破浏览器同源的限制。

### 创建script标签
《高性能JavaScript》中提到了动态脚本元素，作者写道：
1. > 文件在该元素被添加到页面时开始下载。这种技术的重点在于：无论何时启动下载，文件的下载和执行过程不会阻塞页面其他进程。

2. > 使用动态脚本节点下载文件时，返回的代码通常会立刻执行（除了Firefox和Oprea，它们会等待此前所有动态脚本节点执行完毕。）当脚本自执行时，这种机制运行正常。

引用1保证了JSONP请求的时候不会阻塞主线程，引用2保证了JSONP代码在加载完成后，立刻自执行时不会出错。

### callback
服务端在接收到GET请求之后，通常要判断是否有callback参数，如果有，则需要在返回的数据外面加上一个方法名和括号。例如，发起如下请求：
```
http://www.a.com/getSomething?callback=jsonp0
```
那么服务端就会返回如下内容：
```javascript
jsonp0({code:200,data:{}})
```
很明显，由于这是在动态加载的Script标签中包含的内容，那么这就是一段自执行代码，这段代码只有一个函数被调用———jsonp0。  
当然，有执行，则必须先创建，否则就会报错。创建这一步，就需要在调用前执行。  
具体实现如下：
```javascript
function jsonp (url, successCallback, errorCallback, completeCallback) {

    // 声明对象，需要将函数声明至全局作用域
    window.jsonp0 = function (data) {
        successCallback(data);
        if (completeCallback) completeCallback();
    }
    // 创建script标签，并将url后加上callback参数
    var 
        script = document.createElement('script')
        , url = url + (url.indexOf('?') == -1 ? '?' : '&') + 'callback=jsonp0'
    ;
    script.src = url;
    document.head.parentNode.insertBefore(script, document.head);
    // 等到script加载完毕以后，就会自己执行
}
```
上面基本上完成了一个jsonp方法的核心，此时，jsonp0是我们声明好的一个函数，如果服务端正常回传的时候，就会执行jsonp0函数，里面的successCallback回调也会执行。  
### 完善一下
在实际情况下，通常会有许多个jsonp的请求同时调用，
那么既然jsonp0就能满足我们的需要，为什么常常看到jsonp1，jsonp2等等依次累加的代码呢？  
这是因为，请求可能是很多个异步进行。在第一次执行jsonp方法时，window.jsonp0是函数A，此时去加载js文件，在js未加载完毕的情况下，又调用了一次jsonp方法，此时，window.jsonp0指向了函数B。那么等到两次的js加载完毕以后，都会执行第二次的回调。  
所以，我们需要对callback的名字做一个区别处理，累加就能满足需要。  
修改一下代码：
```javascript
var jsonpCounter = 0;
function jsonp (url, successCallback, errorCallback, completeCallback) {
    
    var jsId = 'jsonp' + jsonpCounter++;
    
    // 声明对象，需要将函数声明至全局作用域
    window[jsId] = function (data) {
        successCallback(data);
        if (completeCallback) completeCallback();
        clean();
    }
    // 创建script标签，并将url后加上callback参数
    var 
        script = document.createElement('script')
        , url = url + (url.indexOf('?') == -1 ? '?' : '&') + 'callback=' + jsId
    ;
    script.src = url;
    document.head.parentNode.insertBefore(script, document.head);
    // 等到script加载完毕以后，就会自己执行
    
    //在执行完我们这个方法以后，会有很多script标签出现在head之前，我们需要手动的删除掉他们。
    function clean () {
        script.parentNode.removeChild(script);
        window[jsId] = function(){};
    }
}
```
加入了累加和清理之后，还有一个重要的地方需要处理，就是错误回调。正常来说，我们通常请求jsonp时，会设定一个超时时间，如果超过这个时间以后，就抛出超时异常。  
实现如下：
```javascript
var jsonpCounter = 0;
function jsonp (url, successCallback, errorCallback, completeCallback, timeout) {
    // 略去上面写过的代码
    var 
        timeout = timeout || 10000
        , timer
    ;
    if (timeout) {
        timer = setTimeout(function () {
            if (errorCallback) {
                errorCallback(new Error('timeout'));
            }
            clean();
        }, timeout)
    }
    
    function clean () {
        script.parentNode.removeChild(script);
        window[jsId] = function(){};
        if (timer) clearTimeout(timer);
    }
}
```
这样，基本上就完成了jsonp的全部功能，剩下的可能需要做一些兼容的修改，才算是一个完整的jsonp方法。

## REFER
1. 《高性能JavaScript》  
2. npm上的一个jsonp实现，[JSONP](https://www.npmjs.com/package/jsonp)
