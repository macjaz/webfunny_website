怎样定位前端线上问题，一直以来，都是很头疼的问题，因为它发生于用户的一系列操作之后。错误的原因可能源于机型，网络环境，接口请求，复杂的操作行为等等，在我们想要去解决的时候很难复现出来，自然也就无法解决。 当然，这些问题并非不能克服，让我们来一起看看如何去监控并定位线上的问题吧。 

 

　　背景：市面上的前端监控系统有很多，功能齐全，种类繁多，不管你用或是不用，它都在那里，密密麻麻。往往我需要的功能都在别人家的监控系统里，手动无奈，罢了，怎么才能拥有一个私人定制的前端监控系统呢？做一个自带前端监控系统的前端工程狮是一种怎样的体验呢？

 
　　如果感觉有帮助，或者有兴趣，请[关注](https://zhuanlan.zhihu.com/webfunny) or [Star Me](https://github.com/a597873885/webfunny_monitor) 。

 

　　请移步线上： [前端监控系统](http://www.webfunny.cn/webfunny_multi/home.html)  

 

　　先看一下首页的结果：

　　如果每天都去盯着前端的报错数据，真的很耗费精力，而且很难看出是今天发生的，还是一直存在的报错。

　　所以呢，其实前端项目每天都会有些报错，比如：script error 。我们既不能控制，也不会影响我们的业务，只能让它一直存在。所以我们的前端应用，每天都会有一定数量的报错数据。只要日活量不会波动太大，那么报错数据就会比较平稳，所以我选择跟7天前的报错数据进行比较，如果出现大幅上升，那么就需要我对这个项目进行关注了，而不是每天查看具体的报错数据。

<img src="https://img2018.cnblogs.com/blog/712333/201908/712333-20190802084752835-1692592755.png"/>

　　对于前端应用来说，Js错误的发生直接影响前端应用的质量。对前端异常的监控是整个前端监控系统中的一个重要环节。前端异常包含很多种情况：1. js编译时异常（开发阶段就能排）2. js运行时异常；3. 加载静态资源异常（路径写错、资源服务器异常、CDN异常、跨域）4. 接口请求异常等。这一篇我们只介绍Js运行时异常。

　　监控流程：监控错误 -> 搜集错误 -> 存储错误 -> 分析错误 -> 错误报警-> 定位错误 -> 解决错误

　　首先，我们应该对Js报错情况有个大致的了解，这样才能够及时的了解前端项目的健康状况。所以我们需要分析出一些必要的数据。

　　如：一段时间内，应用JS报错的走势(chart图表)、JS错误发生率、JS错误在PC端发生的概率、JS错误在IOS端发生的概率、JS错误在Android端发生的概率，以及JS错误的归类。

　　然后，我们再去其中的Js错误进行详细的分析，辅助我们排查出错的位置和发生错误的原因。

　　如：JS错误类型、 JS错误信息、JS错误堆栈、JS错误发生的位置以及相关位置的代码；JS错误发生的几率、浏览器的类型，版本号，设备机型等等辅助信息

一、JS Error 监控功能 (数据概览)
 

为了得到这些数据，我们需要在上传的时候将其分析出来。在众多日志分析中，很多字段及功能是重复通用的，所以应该将其封装起来。

    // 设置日志对象类的通用属性
    function setCommonProperty() {
      this.happenTime = new Date().getTime(); // 日志发生时间
      this.webMonitorId = WEB_MONITOR_ID;     // 用于区分应用的唯一标识（一个项目对应一个）
      this.simpleUrl =  window.location.href.split('?')[0].replace('#', ''); // 页面的url
      this.customerKey = utils.getCustomerKey(); // 用于区分用户，所对应唯一的标识，清理本地数据后失效
      this.pageKey = utils.getPageKey();  // 用于区分页面，所对应唯一的标识，每个新页面对应一个值
      this.deviceName = DEVICE_INFO.deviceName;
      this.os = DEVICE_INFO.os + (DEVICE_INFO.osVersion ? " " + DEVICE_INFO.osVersion : "");
      this.browserName = DEVICE_INFO.browserName;
      this.browserVersion = DEVICE_INFO.browserVersion;
      // TODO 位置信息, 待处理
      this.monitorIp = "";  // 用户的IP地址
      this.country = "china";  // 用户所在国家
      this.province = "";  // 用户所在省份
      this.city = "";  // 用户所在城市
      // 用户自定义信息， 由开发者主动传入， 便于对线上进行准确定位
      this.userId = USER_INFO.userId;
      this.firstUserParam = USER_INFO.firstUserParam;
      this.secondUserParam = USER_INFO.secondUserParam;
    }

    // JS错误日志，继承于日志基类MonitorBaseInfo
    function JavaScriptErrorInfo(uploadType, errorMsg, errorStack) {
      setCommonProperty.apply(this);
      this.uploadType = uploadType;
      this.errorMessage = encodeURIComponent(errorMsg);
      this.errorStack = errorStack;
      this.browserInfo = BROWSER_INFO;
    }
    JavaScriptErrorInfo.prototype = new MonitorBaseInfo();

　　封装了一个Js错误对象JavaScriptErrorInfo，用以保存页面中产生的Js错误。其中，setCommonProperty用以设置所有日志对象的通用属性。

　　1）重写window.onerror 方法， 大家熟知，监控JS错误必然离不开它，有人对他进行了测试测试介绍感觉也是比较用心了

　　2）重写console.error方法，为什么要重写这个方法，我不能够给出明确的答案，如果App首次向浏览器注入的Js代码报错了，window.onerror是无法监控到的，所以只能重写console.error的方式来进行捕获，也许会有更好的办法。待window.onerror成功后，此方法便不再需要用了

　　3）重写window.onunhandledrejection方法。 当你用到Promise的时候，而你又忘记写reject的捕获方法的时候，系统总是会抛出一个叫 Unhandled Promise rejection. 没有堆栈，没有其他信息，特别是在写fetch请求的时候很容易发生。 所以我们需要重写这个方法，以帮助我们监控此类错误

    /**
    * 页面JS错误监控
    */
    function recordJavaScriptError() {
      // 重写console.error, 可以捕获更全面的报错信息
      var oldError = console.error;
      console.error = function () {
        // arguments的长度为2时，才是error上报的时机
        // if (arguments.length < 2) return;
        var errorMsg = arguments[0] && arguments[0].message;
        var url = WEB_LOCATION;
        var lineNumber = 0;
        var columnNumber = 0;
        var errorObj = arguments[0] && arguments[0].stack;
        if (!errorObj) errorObj = arguments[0];
        // 如果onerror重写成功，就无需在这里进行上报了
        !jsMonitorStarted && siftAndMakeUpMessage(errorMsg, url, lineNumber, columnNumber, errorObj);
        return oldError.apply(console, arguments);
      };
      // 重写 onerror 进行jsError的监听
      window.onerror = function(errorMsg, url, lineNumber, columnNumber, errorObj)
      {
        jsMonitorStarted = true;
        var errorStack = errorObj ? errorObj.stack : null;
        siftAndMakeUpMessage(errorMsg, url, lineNumber, columnNumber, errorStack);
      };
     function siftAndMakeUpMessage(origin_errorMsg, origin_url, origin_lineNumber, origin_columnNumber, origin_errorObj) {
       var errorMsg = origin_errorMsg ? origin_errorMsg : '';
       var errorObj = origin_errorObj ? origin_errorObj : '';
       var errorType = "";
       if (errorMsg) {
         var errorStackStr = JSON.stringify(errorObj)
         errorType = errorStackStr.split(": ")[0].replace('"', "");
       }
       var javaScriptErrorInfo = new JavaScriptErrorInfo(JS_ERROR, errorType + ": " + errorMsg, errorObj);
       javaScriptErrorInfo.handleLogInfo(JS_ERROR, javaScriptErrorInfo);
     };
   };

OK, 错误日志有了，该怎么计算错误率呢？

　　JS错误发生率 = JS错误个数(一次访问页面中，所有的js错误都算一次)/PV (PC，IOS，Android平台同理)

所以我们需要记下页面的PV记录

    /**
       * 添加一个定时器，进行数据的上传
       * 2秒钟进行一次URL是否变化的检测
       * 10秒钟进行一次数据的检查并上传
       */
      var timeCount = 0;
      setInterval(function () {
        checkUrlChange();
        // 循环5后次进行一次上传
        if (timeCount >= 25) {
          // 如果是本地的localhost, 就忽略，不进行上传

          var logInfo = (localStorage[ELE_BEHAVIOR] || "") +
            (localStorage[JS_ERROR] || "") +
            (localStorage[HTTP_LOG] || "") +
            (localStorage[SCREEN_SHOT] || "") +
            (localStorage[CUSTOMER_PV] || "") +
            (localStorage[LOAD_PAGE] || "") +
            (localStorage[RESOURCE_LOAD] || "");

          if (logInfo) {
            localStorage[ELE_BEHAVIOR] = "";
            localStorage[JS_ERROR] = "";
            localStorage[HTTP_LOG] = "";
            localStorage[SCREEN_SHOT] = "";
            localStorage[CUSTOMER_PV] = "";
            localStorage[LOAD_PAGE] = "";
            localStorage[RESOURCE_LOAD] = "";
            utils.ajax("POST", HTTP_UPLOAD_LOG_INFO, {logInfo: logInfo}, function (res) {}, function () {})
          }
          timeCount = 0;
        }
        timeCount ++;
      }, 200);

　　上边的代码我用了定时器，大概的意思是200毫秒进行一次URL变化的检查，5秒进行一次数据的检查，如果有数据就进行上传，并清空上一次的数据。为什么用定时器呢，因为在单页应用中，路由的切换和地址栏的变化是无法被监控的，我确实没有想到特别好的办法来监控，所以用了这种方式。

　　封装简易的Ajax

　　为了将这些数据上传到我们的服务器，我们总不能每次都用xmlHttpRequest来发送ajax请求吧，所以我们需要自己封装一个简单的Ajax

    /**
     *
     * @param method  请求类型(大写)  GET/POST
     * @param url     请求URL
     * @param param   请求参数
     * @param successCallback  成功回调方法
     * @param failCallback   失败回调方法
     */
    this.ajax = function(method, url, param, successCallback, failCallback) {
      var xmlHttp = window.XMLHttpRequest ? new XMLHttpRequest() : new ActiveXObject('Microsoft.XMLHTTP');
      xmlHttp.open(method, url, true);
      xmlHttp.setRequestHeader('Content-Type','application/x-www-form-urlencoded');
      xmlHttp.onreadystatechange = function () {
        if (xmlHttp.readyState == 4 && xmlHttp.status == 200) {
          var res = JSON.parse(xmlHttp.responseText);
          typeof successCallback == 'function' && successCallback(res);
        } else {
          typeof failCallback == 'function' && failCallback();
        }
      };
      xmlHttp.send("data=" + JSON.stringify(param));
    }

二、JS Error 详细信息解析


 

　　统计JS Error的目的，一、是为了了解线上项目的健康状况，二、是为了分析错误，帮助我们查找问题之所在，并且解决它。

　　所以，如何定位线上的问题，并解决问题，是我们现在要讨论的重点。下面我们需要对几个关键点进行分析：

　　①  某种错误发生的次数——发生次数跟影响用户是成正比的， 如果发生次数跟影响用户数量都很高，那么这是一个比较严重的bug, 需要立即解决。 反之， 如果次数很多，影响用户数量很少。说明这种错误只发生在少量设备中，优先级相对较低，可以择时对该类机型设备进行兼容处理。当然，ip地址访问次数也能说明这个问题

　　

 

　　②  页面发生了哪些错误——这个有利于我们缩小问题的范围，方便我们排查，如：

　　

　　③  错误堆栈——这点不用说，是定位错误最重要的因素。正常情况下，代码都是被压缩的，所以我在后台解析并截取出错代码附近的一部分代码，进行展示，排查错误。PS: 我看到网上有人利用jsMap反向找到代码的具体位置，想法很不错，后期我会加上。 另外，代码虽然被压缩，但是依然很轻松定位到出错的位置，如下图所示， 所以这个功能暂时作为附加题，不用那么着急加上。

　

　　④  设备信息——当错误发生是，分析出用户当时使用设备的浏览器信息，系统版本，设备机型等等，能够帮我们快速的定位到需要兼容的设备，进而提升解决问题的效率。

　　⑤  用户足迹——我个人觉得比较有用，但是代价太高。 因为这个需要记录下用户在页面上的所有行为，需要上传非常多的数据，功能待定。

　　　这个功能已经在后边进行完善了，点击 查看足迹 按钮即可查出这个用的行为足迹，在定位线上问题方面，有很大的作用 , 我在后边的篇幅中有介绍   搭建前端监控系统（五）怎样定位线上问题

　　

 

　　到此，已经收集到了JS错误日志的大部分信息了，并且已经分析出JS错误的详细信息了。

三、JS报错的实时监控与报警
　　既然我们已经具有了搜集js报错和分析报错的能力了，那么我们也可以做到Js报错实时监控，以及实时预警了，这样可以防范线上事故于未然，及时的制止线上事故的持续发生, 减少损失。

 

 

　　如上图所示，我展示了从当前时间向前推算24小时，每小时报错数量。另外展示了7天前同一时间段的报错数量，如果你的项目健康稳定，那么在相同时间段的报错数量应该不会相差太大。如果出现相差太大的情况发生，说明线上出现了问题，此刻应该发出警告，避免线上事故的发生。demo上暂未加上警告功能，但是原理清楚了，后边自然水到渠成。
