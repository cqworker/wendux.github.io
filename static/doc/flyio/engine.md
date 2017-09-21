# Http Engine

Fly引入了Http engine的概念，所谓Http engine，就是真正发起http请求的引擎，这在浏览器中就是XMLHttpRequest或ActiveXObject (IE)，而在node环境中，engine可以用任何能发起网络请求的库／模块实现。 Fly正是通过在不同的环境更换不同的engine而实现同时支持node和browser。

**切换engine**:

```javascript
var  fly=require("flyio")
//浏览器环境下
fly.engine=XMLHttpRequest
//node环境
fly.engine=xx  //任何实现了engine接口的对象
```

上面代码示意了fly如何切换engine，那么如何来提供自定义的engine, 本质上来讲，它就是一个和XMLHttpRequest有相同接口、属性、行为的对象。这很糟糕，因为这必须得了解XMLHttpRequest的各个细节，为了简化engine的实现，Fly提供了一个engine-wrapper模块，它是一个engine的骨架，开发者只需要实现一个适配器（adapter）便可方便的生成一个engine。下面我们看看Fly内置的node engine的大概实现：

```javascript
var  Fly=require("../dist/fly")
var EngineWrapper = require("../dist/engine-wrapper")
//引入fly实现的node adapter
var adapter = require("./adapter/node")
//通过包装node adapter生成一个engine
var engine=EngineWrapper(adapter)
module.exports=new Fly(engine)
```

**engine.setAdapter (adpter)**

每个engine可以随时切换adpter可实现和切换http引擎相同的目的。现在问题变得很简单，就是如何实现adapter.

## Adapter

通过上面的例子可以看出，真正的http请求动作是在adapter中完成的。adapter是一个标准的接口，签名如下

```javascript
function (request, responseCallBack)
```

**request**: http请求信息，由engine 传给adapter，**基本结构字段**如下：

```javascript
{
  method:"",//请求方法， GET 、POST ...
  headers:{},//请求头
  url:"",//请求地址
  timeout:"",//超时时间
  data:""//请求数据，GET请求时为null
}
```

**responseCallBack(response)**: 响应回调

请求结束时adapter应调用此函数，通知engine请求结束, response**基本结构字段**如下：

```javascript
{
    responseText: '{"aa":5}',//响应内容，为字符串
    statusCode: 200,// http 状态码，发生异常时，值为0
    errMsg:"", //错误信息，
    headers: {}//响应头
}
```

整个请求过程为：每次请求开始的时候，fly将用户本次的请求信息传给http engine，然后http engine根据用户请求信息生成一个request对象传递给adapter， 接着adapter发起真正的http请求，等到请求结束时，adapter通过调用responseCallBack将请求结果返回给http engine.  然后http engine将结果返回给fly。

**基本结构字段：**

所谓基本结构字段是fly定义的标准字段。除了这些基本结构字段，可以任意扩展：

对于request对象， 用户在发起的请求配置options中的其它字段也会merge到request对象中，这样就可以在adapter 中获取到，这在自定义adapter时非常有用。

对于response对象，可以在adapter 中给其添加任何自定义属性（但不能是添加key为“data”的字段, 因为在fly中data永远代表请求的最终结果，EngineWrapper中会自动设置），然后上层在then回调中可以取出。

### 一个简单的例子

```javascript
var engine= EngineWrapper(function (request,responseCallback) {
            responseCallback({
                statusCode:200,
                responseText:"你变或者不变，我都不变😜。",
                extraFeild:"自定义字段"
            })
        })
fly.engine=engine;

fly.get("../package.json").then(d=>{
    log(d.data)
    log(d.extraFeild)
})

控制台输出

> 你变或者不变，我都不变😜。
> 自定义字段

```

这个例子中，adapter 并没有真正发起 http 请求，而是直接返回了固定内容，这样 fly 上层请求任何借口收到的内容永远都是指定的 responseText 的内容。下面，我们看看真正强大的使用姿势。



## 请求重定向

 如果你开发的是一个 PC桌面应用、或者一个APP，你需要以通过内嵌 h5 页面的方式实现部分功能，这类应用我们统一称之为混合应用。然后你会发现你**无法干涉 ** webview 中 h5 页面发起的 ajax 请求，无法进行统一的请求管理。为什么要进行统一的请求管理呢？主要有以下几种场景：

1. cookie 同步.
2. 接口安全
3. 访问控制
4. 性能
5. 缓存

### Cookie同步

混合APP种非常关键的一点就是 Native 和 h5 之间的 cookie 同步。因为 Native 和 h5 都可以发起网络请求，而两者所用的请求池并不是同一个（h5  ajax 请求是 webview 发起的），这就会导致 Native 和 h5 cookie 不能同步，如果不能统一管理请求，就会导致你要在不同的场景做大量的同步工作。更沮丧的是，在ios系统中WKWebview并没有提供 cookie 同步的接口，开发者通常是通过注入 javascript的方式间接实现，但是这种做法有个缺点，就是无法同步 httpOnly 属性的 cookie , 因为 javascript 无法操作具有httpOnly属性的cookie ( 这主要是因为现代浏览器为了防止xss攻击成功后确保用户cookie不会被恶意脚本窃取而添加的特性 )。

### 接口安全

为了网络传输数据安全，https协议已经越来越普及，但是，浏览器/webview 对于https请求，默认的证书校验策略是先查看本地信任证书列表，如果没有，则去检验CA链，这也就意味着，如果攻击者通过伪造的证书开一个代理服务器，然后在自己的手机中手动添加这个伪造证书至本地信任列表， 然后攻击者将手机代理指向其代理服务器，那么接口数据将会完全暴漏在攻击者面前。而目前防止代理方式攻击方式就是在端上进行证书校验，这可以保证Native发起的请求数据是安全的，但是h5通过webview发起的请求仍将会暴漏，如果你的APP是一个金融理财类的应用，这将非常危险。

### 访问控制

由于浏览器的**同源策略**，如果h5请求需要**跨域**，是一件比较麻烦的事。现在的主流的跨域方案，无论是jsonp，还是通过设置**CORS**（Cross Origin Resource Sharing）相关的请求头，都需要服务端支持。然而很多时候我们需要能在端上指定哪些域可以访问，哪些域不能访问，如果请求在webview中发起，端上根本无法干涉。

### 性能

不同webview，底层对网络请求处理策略往往会有不同，有的是采用一个线程池，有的是每次请求都会创建一个新的线程，这就导致在有些系统上原生webview的请求会相对较慢。

### 缓存

现在有些混合app为了实现页面秒开，都采用一些缓存自管理的方案，常见的就是将线上新版本的h5页面提前打包，然后在没次APP启动后检测更新、下载。然后拦截 webview 的所有请求，对于静态资源，直接从下载好的文件中读取，而动态数据则通过ajax去后台动态拉去，如果只是这样，那倒没什么问题，但是，有时有些静态资源也是需要通过ajax去拉取，比如，一个配置 json， 又或是一个 md格式的帮助文档。如果不能拦截ajax请求，也就意味着这些静态资源还是要通过网络去服务端拉取。



综上所属，如果能在端上进行统一的请求管理，上面这些问题将会引刃而解。但是，如何实现在端上进行统一的请求管理。我们说过，目前对于大多数平台的webview，它们发起的 ajax请求，native都是无法直接干涉的。那么现在有了Fly，我们应该如何解决这个问题？



## 远程Http Engine

我们说过，在浏览器环境中，fly使用的默认的engine就是 XMLHttpRequest。 如果能在Native上实现一个engine，然后供浏览器中的fly使用，那么也就会将原本应该在浏览器中发起的请求重定向到了Native上。而这个在Native上实现的engine，我们称其为为远程engine，这是因为调用者和执行者并不在同一个环境。我们看看在fly中如何使用远程engine。在介绍这个之前，我们先来了解一下什么是 **Javascript Bridge** 。 



> Javascript Bridge 是指web应用中Javascript与Native之间接口互调，数据传输的一个桥梁。现在github中有一些成熟易用的移动端跨平台实现如: [dsBridge](https://github.com/wendux/DSBridge-Android) 、 [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 。

这样的话，我们可以在adapter 将请求通过 javascript bridge 转发到 Natvie上，然后在native 上进行统一的网络请求，这样就可以在native进行统一的证书验证、cookie管理、访问控制 还有缓存策略。等到native请求完成后再将请求结果传递给adapter。 然后adapter再传递给fly。现在问题简化为，只需为不同的 javascript bridge提供一个adapter即可。为了方便使用，fly预置了DSBridge的适配器 。

### Native实现

fly将http请求重定向到Native时，Native需要实现真正的http请求。具体实现由端上根据使用的网络框架自定。Fly官方提供了Android端的实现示例，可供开发者参考。

### DSBridge Adapter

[dsBridge](https://github.com/wendux/DSBridge-Android)  是一个优秀的跨平台的 Javascript Bridge ，最大的特点是不仅支持**异步调用**，也支持**异步调用**和进度**连续调用**。如果你的 App 使用的是DSBridge， 那么你可以非常方便的使用fly。

```javascript
var adapter = require("flyio/dist/adapter/dsbridge")
var EngineWrapper = require("flyio/dist/engine-wrapper")
var Fly = require("flyio/dist/fly")

var dsEngine = EngineWrapper(adapter)
var fly = new Fly(engine);
//然后你就可以使用fly发起请求了
fly.get("xxx.com")...
```



现在在h5中通过fly发起的所有ajax请求都会被重定向到端上，下一节我们看看Native端改如何实现。

