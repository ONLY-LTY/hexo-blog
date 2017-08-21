---
title: Cross Domin
date: 2016-11-09 14:10:10
tags:
  - 前端
  - 跨域
photos:
  - /img/20161114.jpeg
---
一次跨域之旅
<!--more-->

&emsp;&emsp;小洋前几天帮同学写了一个短信验证码的H5页面，应对他的奇葩需求，小洋在页面里面用Jquery直接调用的第三方的发短信的接口。当点击发送的时候，发现jQuery进入的是Error，但是一看手机已经收到验证码了。小洋也是相当的郁闷。然后F12一番，跨域了。小洋闭目三思，本地ajax请求第三方肯定跨域了，可是收到验证码了说明请求成功了啊。小洋以前有需求的时候戏玩Js。然不知其所以然，这次看了看jQuery的ajax部分的源码。才恍然大悟。

### 跨域
&emsp;&emsp;跨域这个概念就不多说了，网上一大堆。说白了就是协议、域名、端口有任何一个不同，都被当做是不同的域。比如我本地http://localhost:8080/index.html 中通过ajax去请求 http://localhost:9999/example 是不允许的，因为端口不同，是不同的域。这就是跨域访问。

### JSONP
&emsp;&emsp;解决跨域的方法有很多。本文只是针对我上面提的问题说说 *jQuery* 中用到的 *JSONP*。大家不要胡乱猜想 *JSONP* 和 *JSON* 是两完全不一样的东西。*JSON* 是一种数据交互的格式。而JSONP是根据开发人员想出来非官方的一种跨域数据交互协议。其实我理解为是利用了html标签的一个小漏洞。那么为什么 *JSONP* 能跨域呢。我们知道【script】 【img】 【iframe】这个几个标签是是可以指定src属性从任何地方获取资源的。下面演示一下:

```html
<!DOCTYPE HTML>
<html >
<head>
  <title></title>
  <script type="text/javascript">
  var localHandler = function(data){
      alert('我是本地函数，可以被跨域调用，远程带来的数据是：' + data.result);
  };
  </script>
  <script type="text/javascript" src="http://remoteserver.com/data"></script>
</head>
<body>

</body>
</html>
```
&emsp;&emsp;远程服务端返回的数据是这样的，就是一个字符串。

```string
localHandler({"result":"我是远程js带来的数据"});
```
&emsp;&emsp;上面我们可以知道当我们加载页面的时候回去请求http://remoteserver.com/data 这个接口，接口返回了一段js代码。然后浏览器就会执行这段js代码。看的出来，服务端返回的js代码调用了我们的localHandler方法，并且传入数据。这个数据就是我们正常服务端接口返回的数据。现在要支持 *JSONP* 跨域。服务端需要将数据进行一下包装。类似上面那样的。这就是 *JSONP* 跨域的原理。

&emsp;&emsp;那么问题来了，服务端是怎么知道我本地需要调用那个JS方法的。针对这个问题我们可以动态的生成【script】标签，演示如下:

```html
<!DOCTYPE html>
<html >
  <head>
    <title></title>
    <script type="text/javascript">
    // 得到航班信息查询结果后的回调函数
    var flightHandler = function(data){
        alert('你查询的航班结果是：票价 ' + data.price + ' 元，' + '余票 ' + data.tickets + ' 张。');
    };
    // 提供jsonp服务的url地址（不管是什么类型的地址，最终生成的返回值都是一段javascript代码）
    var url = "http://remoteserver.com/jsonp/data?code=CA1998&callback=flightHandler";
    // 创建script标签，设置其属性
    var script = document.createElement('script');
    script.setAttribute('src', url);
    // 把script标签加入head，此时调用开始
    document.getElementsByTagName('head')[0].appendChild(script);
    </script>
  </head>
  <body>

  </body>
</html>

```
&emsp;&emsp;上面我们通过动态的生成【script】标签，并且将本地的调用方法以callback参数传给后台。后台需要获取这个参数组装js代码。其实JQuery内部实现原理都大同小异。它帮你做了很多事情。

### 结束语
&emsp;&emsp;Jquery 中的ajax使用了jsonp这样的技术，最开始上面的问题，由于我用了Jquery的ajax去请求第三方发短信接口，Jquery发现是跨域请求就默认使用了jsonp的方式并且使用的默认的回调方法，由于第三方接口没有配合jsonp的使用，返回的不是可调用的js代码，Jquery就是处理的时候进入到了error回调。这就是为啥接口调用成功了但是进入的error的回调。知道了jsonp的原理，我们可以很容易的看出这样只能支持GET请求，对于POST请求时无能为力的。这也是jsonp的一个限制吧。在Jquery的高版本中，会将跨域的POST请求，自动转换成GET请求。如果发现自己method写的POST然后发现成功了，不要惊讶，打开浏览器F12查看一下你就会发现其实是GET请求了。<font color='red'>再有使用jsonp一定要服务端的配合使用，不要自己在前端写写就觉得可以实现跨域了。</font>现在基本上都用Nginx做反向代理来解决跨域问题，很方便。

&emsp;&emsp;通过这次小洋知道了跨域的一系列原理，文章内容不入法眼，只是提醒一下自己，凡是要知其所以然才能随心所欲。

---
<p align='center'><font color='blue'>与其临渊羡鱼,不如退而结网</font></p><p align='right'>--《史记·汉书·董仲舒传》</p>

---
