---
title: WebViewJavascriptBridge源码剖析
date: 2017-10-10 19:25:05
tags:
---

对于任意hybrid APP，不可避免进行native与web之间的交互。[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) 就是一款用于实现原生端与web端无缝交互的三方库，应用广泛，支持UIWebView、WKWebView（iOS）以及WebView（OSX），原理一致，本文借助OC的UIWebView进行分析。

## 框架简介

所谓交互，无非就是两端（native端与JS端）能够互相调用方法，并传递数据。iOS7之后，随着JavaScriptCore框架的推出，为native与JS的交互铺平了道路，使用该框架即可实现OC与JS的互调。而在iOS7之前，OC与jS的交互只能借助于WebView，OC调用JS依赖于WebView提供的执行JS的方法，而JS调用OC则需要采用URL拦截的方式。WebViewJavascriptBridge就是为WebView中的JS交互而生的，采用后者的交互方式。

大致原理为OC端和JS端各自保存一个bridge对象，并各自维护开放给另一端调用的方法集合以及回调方法集合，两端之间的交互通过传递handleName（可以理解为方法id）以及callBackId来实现方法调用以及回调的。

代码文件结构如下（V6.0.2版本）：

- **WebViewJavascriptBridge**

  基于UIWebView/WebView的OC端交互逻辑处理类，面向OC业务层，提供了注册OC方法、调用JS方法等接口。

- **WebViewJavascriptBridgeBase**

  OC端bridge对象基础服务类，维护OC端开放给JS端的方法以及OC回调方法，实现OC向JS发送数据的具体逻辑。

- **WebViewJavascriptBridge_JS**

  维护了一份JS代码，用于JS环境的注入。同时维护JS端的bridge对象，管理JS端注册的方法集合以及回调方法集合，面向Web端提供注册JS方法、调用OC端方法的接口。

- **WKWebViewJavascriptBridge**

  基于WKWebView的OC端交互逻辑处理类，职能同WebViewJavascriptBridge。

- ExampleApp.html

  非框架文件，由于我们借助WebViewJavascriptBridge的官方example讲解，该文件作为模拟开发环境的web页面示例。

## Bridge环境初始化

#### OC端bridge初始化

业务层创建好webView实例之后，需要根据此webView创建对应的Bridge，即初始化bridge对象：

```objectivec
//初始化OC端Bridge对象
_bridge = [WebViewJavascriptBridge bridgeForWebView:webView];
//设置代理，避免框架对webView代理的入侵
[_bridge setWebViewDelegate:self];
```

相关实现如下（仅保留相关代码）：

```objectivec
+ (instancetype)bridgeForWebView:(id)webView {
    return [self bridge:webView];
}

+ (instancetype)bridge:(id)webView {
    if ([webView isKindOfClass:[WVJB_WEBVIEW_TYPE class]]) {
        WebViewJavascriptBridge* bridge = [[self alloc] init];
        [bridge _platformSpecificSetup:webView];
        return bridge;
    }
    return nil;
}

- (void) _platformSpecificSetup:(WVJB_WEBVIEW_TYPE*)webView {
    _webView = webView;
    _webView.delegate = self;
    _base = [[WebViewJavascriptBridgeBase alloc] init];
    _base.delegate = self;
}

//WebViewJavascriptBridgeBase实例初始化
//messageHandlers：保存OC端注册给JS端的方法集合，key为handleName，value为方法实现block
//startupMessageQueue：由于OC端调用JS方法时，JS环境可能还没有初始化好，该数组暂存JS环境初始化之前的JS方法调用操作（保存调用的handleName）
//responseCallbacks：保存OC端调用JS方法的回调block，key为callBackId，value会回调实现block
//_uniqueId：表示消息的唯一性
- (id)init {
    if (self = [super init]) {
        self.messageHandlers = [NSMutableDictionary dictionary];
        self.startupMessageQueue = [NSMutableArray array];
        self.responseCallbacks = [NSMutableDictionary dictionary];
        _uniqueId = 0;
    }
    return self;
}
```

OC端的bridge初始化即为webView实例创建了一个bridge对象，并初始化了相关的数据结构。

#### JS端Bridge初始化

JS端bridge对象的初始化要比OC端复杂得多，因为JS环境的初始化也是在原生端完成的，涉及原生端对webView初始化JS环境操作指令的捕捉，以及JS代码的注入。

webView加载时，执行了如下JS：

```javascript
function setupWebViewJavascriptBridge(callback) {
    if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
    if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
    window.WVJBCallbacks = [callback];
    var WVJBIframe = document.createElement('iframe');
    WVJBIframe.style.display = 'none';
    WVJBIframe.src = 'https://__bridge_loaded__';
    document.documentElement.appendChild(WVJBIframe);
    setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
}
```

传入的参数是一个函数，暂且不管该函数的内容。首先判断当前环境中是否存在WebViewJavascriptBridge对象，若存在则直接调用传入的函数并已WebViewJavascriptBridge作为参数。该WebViewJavascriptBridge就是JS端的bridge对象，页面首次加载时，JS环境没有初始化好，也就是该Bridge对象没有创建好，因此会先将传入的函数暂存在数组WVJBCallbacks中。

重点看 ``WVJBIframe.src = 'https://__bridge_loaded__';`` ，iframe可以理解为webView的窗口，当改变iframe的src时，页面就会进行刷新并加载指定的URL，此处试图加载 ``https://__bridge_loaded__``。

我们知道，UIWebView刷新页面前，会先回调 ``shouldStartLoadWithRequest`` 方法。因此可以在该代理方法中拦截该URL的加载，并执行相应的逻辑。可以在 WebViewJavascriptBridge.m 文件中看到如下代码：

```objectivec
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    if (webView != _webView) { return YES; }
    
    NSURL *url = [request URL];
    __strong WVJB_WEBVIEW_DELEGATE_TYPE* strongDelegate = _webViewDelegate;
    
    //判断是否是WebViewJavascriptBridge内部发起的请求
    if ([_base isWebViewJavascriptBridgeURL:url]) {
        //判断是否是JS环境初始化请求
        if ([_base isBridgeLoadedURL:url]) {
            [_base injectJavascriptFile];
        }
        //判断是否是web端交互请求
        else if ([_base isQueueMessageURL:url]) {
            NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
            [_base flushMessageQueue:messageQueueString];
        }
        //未知请求
        else {
            [_base logUnkownMessage:url];
        }
        return NO;
    }
    //常规webView加载请求，交给业务层的webView自行处理
    else if (strongDelegate && [strongDelegate respondsToSelector:@selector(webView:shouldStartLoadWithRequest:navigationType:)]) {
        return [strongDelegate webView:webView shouldStartLoadWithRequest:request navigationType:navigationType];
    } else {
        return YES;
    }
}
```

重点看中间部分，主要拦截了两种请求：JS环境初始化请求和web端交互请求，都是由WebViewJavascriptBridge内部发起的。

```objectivec
#define kOldProtocolScheme @"wvjbscheme"
#define kNewProtocolScheme @"https"
#define kQueueHasMessage   @"__wvjb_queue_message__"
#define kBridgeLoaded      @"__bridge_loaded__"

//判断是否是WebViewJavascriptBridge内部发起的请求
- (BOOL)isWebViewJavascriptBridgeURL:(NSURL*)url {
    if (![self isSchemeMatch:url]) {
        return NO;
    }
    return [self isBridgeLoadedURL:url] || [self isQueueMessageURL:url];
}

//判断请求的协议是否符合WebViewJavascriptBridge的约定
- (BOOL)isSchemeMatch:(NSURL*)url {
    NSString* scheme = url.scheme.lowercaseString;
    return [scheme isEqualToString:kNewProtocolScheme] || [scheme isEqualToString:kOldProtocolScheme];
}

//判断是否是WebViewJavascriptBridge发起的web交互请求
- (BOOL)isQueueMessageURL:(NSURL*)url {
    NSString* host = url.host.lowercaseString;
    return [self isSchemeMatch:url] && [host isEqualToString:kQueueHasMessage];
}

//判断是否是WebViewJavascriptBridge发起的JS环境初始化请求
- (BOOL)isBridgeLoadedURL:(NSURL*)url {
    NSString* host = url.host.lowercaseString;
    return [self isSchemeMatch:url] && [host isEqualToString:kBridgeLoaded];
}
```

因此，之前页面中加载的 ``https://__bridge_loaded__`` 就会被拦截，并识别为JS环境初始化请求，由OC端执行相应的的初始化逻辑。

```objectivec
- (void)injectJavascriptFile {
    NSString *js = WebViewJavascriptBridge_js();
    [self _evaluateJavascript:js];
    if (self.startupMessageQueue) {
        NSArray* queue = self.startupMessageQueue;
        self.startupMessageQueue = nil;
        for (id queuedMessage in queue) {
            [self _dispatchMessage:queuedMessage];
        }
    }
}
```

该方法中执行了WebViewJavascriptBridge_JS文件中的js代码。为方便阅读，我把JS代码拎出来如下：

```javascript
;(function() {
	if (window.WebViewJavascriptBridge) {
		return;
	}
	if (!window.onerror) {
		window.onerror = function(msg, url, line) {
			console.log("WebViewJavascriptBridge: ERROR:" + msg + "@" + url + ":" + line);
		}
	}
	//创建Bridge对象
	window.WebViewJavascriptBridge = {
		registerHandler: registerHandler,
		callHandler: callHandler,
		disableJavscriptAlertBoxSafetyTimeout: disableJavscriptAlertBoxSafetyTimeout,
		_fetchQueue: _fetchQueue,
		_handleMessageFromObjC: _handleMessageFromObjC
	};

	//初始化一些变量
	var messagingIframe;
	//消息队列，存放发送给OC的数据
	var sendMessageQueue = [];
	//存放JS端注册的函数
	var messageHandlers = {};
	
	var CUSTOM_PROTOCOL_SCHEME = 'https';
	var QUEUE_HAS_MESSAGE = '__wvjb_queue_message__';
	
	//存放回调函数
	var responseCallbacks = {};
	//标识消息的唯一性
	var uniqueId = 1;
	//是否异步执行
	var dispatchMessagesWithTimeoutSafety = true;

	//注册给OC调用的方法
	function registerHandler(handlerName, handler) {
		messageHandlers[handlerName] = handler;
	}
	
	//调用OC方法
	function callHandler(handlerName, data, responseCallback) {
		if (arguments.length == 2 && typeof data == 'function') {
			responseCallback = data;
			data = null;
		}
		_doSend({ handlerName:handlerName, data:data }, responseCallback);
	}

	//禁止异步发送消息
	function disableJavscriptAlertBoxSafetyTimeout() {
		dispatchMessagesWithTimeoutSafety = false;
	}

	//向OC发送消息
	function _doSend(message, responseCallback) {
		if (responseCallback) {
            var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
            //responseCallBacks对象对用callBackId存放回调函数
			responseCallbacks[callbackId] = responseCallback;
			message['callbackId'] = callbackId;
		}
		sendMessageQueue.push(message);
		messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
	}

	//获取当前消息队列待发送的消息
	function _fetchQueue() {
		var messageQueueString = JSON.stringify(sendMessageQueue);
		sendMessageQueue = [];
		return messageQueueString;
	}

	//接收从OC发来的消息
	function _dispatchMessageFromObjC(messageJSON) {
		if (dispatchMessagesWithTimeoutSafety) {
			setTimeout(_doDispatchMessageFromObjC);
		} else {
			 _doDispatchMessageFromObjC();
		}
		
		function _doDispatchMessageFromObjC() {
			var message = JSON.parse(messageJSON);
			var messageHandler;
			var responseCallback;

			if (message.responseId) {
				responseCallback = responseCallbacks[message.responseId];
				if (!responseCallback) {
					return;
				}
				responseCallback(message.responseData);
				delete responseCallbacks[message.responseId];
			} else {
				if (message.callbackId) {
					var callbackResponseId = message.callbackId;
					responseCallback = function(responseData) {
						_doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
					};
				}
				
				var handler = messageHandlers[message.handlerName];
				if (!handler) {
					console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
				} else {
					handler(message.data, responseCallback);
				}
			}
		}
	}
	
	function _handleMessageFromObjC(messageJSON) {
        _dispatchMessageFromObjC(messageJSON);
	}

	messagingIframe = document.createElement('iframe');
	messagingIframe.style.display = 'none';
	messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
	document.documentElement.appendChild(messagingIframe);

	registerHandler("_disableJavascriptAlertBoxSafetyTimeout", disableJavscriptAlertBoxSafetyTimeout);
	
	//Bridge初始化完毕后，执行webView加载时待执行的JS代码
	setTimeout(_callWVJBCallbacks, 0);
	function _callWVJBCallbacks() {
		var callbacks = window.WVJBCallbacks;
		delete window.WVJBCallbacks;
		for (var i=0; i<callbacks.length; i++) {
			callbacks[i](WebViewJavascriptBridge);
		}
	}
})();
```

分析一下，所做的事情很简单，就是创建了一个bridge对象，该对象维护了若干函数，OC端可以通过bridge对象获取到相应的函数来执行调用。

- registerHandler

  用于JS端注册供OC端调用的方法

- callHandler

  用于调用OC方法

- disableJavscriptAlertBoxSafetyTimeout

  关闭消息异步发送

- _fetchQueue

  获取当前JS端待发送的消息

- _handleMessageFromObjC

  接收并处理OC端发送给JS端的数据

另外，代码最后执行了``_callWVJBCallbacks`` 函数 ，还记得之前webView加载时暂存在WVJBCallbacks的函数吗？那时因为bridge未初始化先暂存起来，现在bridge初始化完毕了再来执行。

至此JS环境初始化完成。不过上面 ``injectJavascriptFile`` 方法中除了执行JS代码，还执行了如下代码：

```objectivec
if (self.startupMessageQueue) {
    NSArray* queue = self.startupMessageQueue;
    self.startupMessageQueue = nil;
    for (id queuedMessage in queue) {
        [self _dispatchMessage:queuedMessage];
    }
}
```

startupMessageQueue是OC端bridge维护的属性，之前介绍过其用于暂存JS环境初始化之前OC发起的JS消息调用。现在JS环境初始化好了，便可取出其中的消息发送了。

## OC调用JS

OC调用JS可以分为4个过程：

1. JS注册方法给OC
2. OC调用JS注册的方法
3. JS方法被调用，执行自身逻辑
4. JS回调数据给OC

#### JS注册方法

毫无疑问，这一步应该在webView加载时执行。还记得ExampleApp.html在加载时传了一个函数给 ``setupWebViewJavascriptBridge()`` 吗。这个函数后面被执行，函数内容就脑阔相关JS方法的注册。示例中该函数相关内容如下：

```javascript
function(bridge) {
	...
    //注册testJavascriptHandler方法
    bridge.registerHandler('testJavascriptHandler', function(data, responseCallback) {
        log('ObjC called testJavascriptHandler with', data)
        var responseData = { 'Javascript Says':'Right back atcha!' }
        log('JS responding with', responseData)
        responseCallback(responseData)
    })
	...
}
```

没错，就是通过调用bridge对象的registerHandler方法来注册的。实现很简单：

```javascript
function registerHandler(handlerName, handler) {
    messageHandlers[handlerName] = handler;
}
```

单纯只是把handleName及方法实现绑定在一起，保存在messageHandlers中。

#### OC发起调用

OC端通过JS注册的handleName调用JS相应的方法，同时可以设置回调操作。

```objectivec
[_bridge callHandler:@"testJavascriptHandler" data:@{ @"foo":@"before ready" } responseCallback:^(id responseData) {
    NSLog(@"testJavascriptHandler callBack: %@", responseData);
}];
```

```objectivec
- (void)callHandler:(NSString *)handlerName data:(id)data responseCallback:(WVJBResponseCallback)responseCallback {
    [_base sendData:data responseCallback:responseCallback handlerName:handlerName];
}

- (void)sendData:(id)data responseCallback:(WVJBResponseCallback)responseCallback handlerName:(NSString*)handlerName {
    NSMutableDictionary* message = [NSMutableDictionary dictionary];
    if (data) {
        message[@"data"] = data;
    }
    //若OC端需要回调，则创建唯一的callbackId保存该回调block
    if (responseCallback) {
        NSString* callbackId = [NSString stringWithFormat:@"objc_cb_%ld", ++_uniqueId];
        self.responseCallbacks[callbackId] = [responseCallback copy];
        message[@"callbackId"] = callbackId;
    }
    if (handlerName) {
        message[@"handlerName"] = handlerName;
    }
    //OC端将数据封装成特定格式的字典，JS端在拿到数据后按照该格式解封装
    [self _queueMessage:message];
}

- (void)_queueMessage:(WVJBMessage*)message {
    if (self.startupMessageQueue) {
        //若startupMessageQueue不为空，说明JS环境未初始化，暂存消息
        [self.startupMessageQueue addObject:message];
    } else {
        //直接发送消息
        [self _dispatchMessage:message];
    }
}

- (void)_dispatchMessage:(WVJBMessage*)message {
    NSString *messageJSON = [self _serializeMessage:message pretty:NO];
    [self _log:@"SEND" json:messageJSON];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\\" withString:@"\\\\"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\"" withString:@"\\\""];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\'" withString:@"\\\'"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\n" withString:@"\\n"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\r" withString:@"\\r"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\f" withString:@"\\f"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2028" withString:@"\\u2028"];
    messageJSON = [messageJSON stringByReplacingOccurrencesOfString:@"\u2029" withString:@"\\u2029"];
    
    //序列化消息数据并进行转义后，通过webView的stringByEvaluatingJavaScriptFromString执行JS代码。JS内容为调用bridge对象的_handleMessageFromObjC方法，并传入序列化后的数据
    NSString* javascriptCommand = [NSString stringWithFormat:@"WebViewJavascriptBridge._handleMessageFromObjC('%@');", messageJSON];
    if ([[NSThread currentThread] isMainThread]) {
        [self _evaluateJavascript:javascriptCommand];
    } else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            [self _evaluateJavascript:javascriptCommand];
        });
    }
}
```

#### JS接收调用

JS通过bridge的 ``_handleMessageFromObjC`` 方法接收OC的消息，具体实现如下：

```javascript
function _handleMessageFromObjC(messageJSON) {
    _dispatchMessageFromObjC(messageJSON);
}

function _dispatchMessageFromObjC(messageJSON) {
    //根据dispatchMessagesWithTimeoutSafety标志量决定是否异步执行
    if (dispatchMessagesWithTimeoutSafety) {
        setTimeout(_doDispatchMessageFromObjC);
    } else {
         _doDispatchMessageFromObjC();
    }
    function _doDispatchMessageFromObjC() {
        var message = JSON.parse(messageJSON);
        var messageHandler;
        var responseCallback;

        //若responseId不为空，说明这是一个回调消息
        if (message.responseId) 
        {
            //从responseCallbacks中取出事先保存好的回调函数执行
            responseCallback = responseCallbacks[message.responseId];
            if (!responseCallback) {
                return;
            }
            responseCallback(message.responseData);
            //执行完毕后移除回调函数
            delete responseCallbacks[message.responseId];
        } 
        //若responseId不存在，说明这是一个OC发起的消息
        else 
        {
            //若callbackId不为空，说明需要回调OC
            if (message.callbackId) {
                var callbackResponseId = message.callbackId;
                //创建回调函数，传入JS方法中，由web业务方决定何时执行该回调函数来回调OC
                responseCallback = function(responseData) {
                    //通过调用_doSend向OC发送包含responseId的数据，来指明这是一个回调调用，responseId的值为OC传过来的callbackId。
                    _doSend({ handlerName:message.handlerName, responseId:callbackResponseId, responseData:responseData });
                };
            }
            //从messageHandlers中根据handlerName取出之前注册的方法
            var handler = messageHandlers[message.handlerName];
            if (!handler) {
                console.log("WebViewJavascriptBridge: WARNING: no handler for message from ObjC:", message);
            } else {
                //执行JS方法自身逻辑
                handler(message.data, responseCallback);
            }
        }
    }
}
```

首先解析OC传过来的数据，然后判断数据中是否存在responseId字段，若存在，则被认为这是一个OC回调的消息，否则被认为是OC发起的消息。此处不存在responseId，因为当前场景为OC发起的消息。但是因为消息数据中含有callbackId，被认为是OC需要回调，因此创建一个callBack函数，函数内容为调用 ``_doSend`` 向OC发送指定格式的数据（重点是包含responseId字段，OC端会根据此字段识别这是一个JS回调调用，同JS端）。随后，根据OC传过来的handleName获取到注册的方法，传入data及callBack函数来调用。至此，web端的方法得到调用：

```javascript
bridge.registerHandler('testJavascriptHandler', function(data, responseCallback) {
    log('ObjC called testJavascriptHandler with', data)
    var responseData = { 'Javascript Says':'Right back atcha!' }
    log('JS responding with', responseData)
    responseCallback(responseData)
})
```

#### OC回调

JS注册的方法被调用的时候，接收了一个回调OC的函数，业务端根据实际情况选择合适的时机调用该函数以回调数据给OC端。关于该回调函数的创建上面已经有说明，回调的方式就是向OC发送消息，属于JS调用OC的范畴，将在下面介绍。

## JS调用OC

和OC调用JS一样，JS调用OC也是同样的4个过程：

1. OC注册方法给JS
2. JS发起调用
3. OC接收调用并执行相应逻辑
4. OC回调数据给JS

#### OC注册方法

和JS类似，OC端也是通过bridge对象注册方法。

```objectivec
[_bridge registerHandler:@"testObjcCallback" handler:^(id data, WVJBResponseCallback responseCallback) {
    NSLog(@"testObjcCallback called: %@", data);
    responseCallback(@"Response from testObjcCallback");
}];
```

实现也一致，绑定handleName与block，保存到bridge的messageHandlers属性中：

```objectivec
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```

#### JS发起调用

web业务端根据实际场景调用OC，示例中是在ExampleApp.html加载时向DOM中添加了一个按钮元素，点击按钮调用OC注册的方法。

```javascript
var callbackButton = document.getElementById('buttons').appendChild(document.createElement('button'))
callbackButton.innerHTML = 'Fire testObjcCallback'
callbackButton.onclick = function(e) {
    e.preventDefault()
    log('JS calling handler "testObjcCallback"')
    bridge.callHandler('testObjcCallback', {'foo': 'bar'}, function(response) {
        log('JS got response', response)
    })
} 
```

可以看到，JS端通过bridge对象的 ``callHandler`` 执行调用。和OC调用JS很类似，也是传了3个参数：handleName、data、回调函数。相关方法实现如下：

```javascript
function callHandler(handlerName, data, responseCallback) {
    if (arguments.length == 2 && typeof data == 'function') {
        responseCallback = data;
        data = null;
    }
    _doSend({ handlerName:handlerName, data:data }, responseCallback);
}

function _doSend(message, responseCallback) {
    if (responseCallback) {
        var callbackId = 'cb_'+(uniqueId++)+'_'+new Date().getTime();
        //responseCallBacks对象对用callBackId存放回调函数
        responseCallbacks[callbackId] = responseCallback;
        message['callbackId'] = callbackId;
    }
    sendMessageQueue.push(message);
    messagingIframe.src = CUSTOM_PROTOCOL_SCHEME + '://' + QUEUE_HAS_MESSAGE;
}
```

可以看到最终执行的是 ``_doSend`` 函数，这个函数之前在讲解JS回调OC的时候也遇到过，其作用就是向OC发送消息。``_doSend`` 执行时，若JS需要回调，即responseCallback参数不为空，则会创建唯一callbackId，并保存回调函数。随后，将数据封装成指定格式的对象（格式同OC调用JS时一致），发送给OC。此处，我们又见到了眼熟的src，之前在web端初始化JS环境的时候遇到过。没错，JS无法直接调用OC，此处依旧是通过URL拦截的方式实现的。首先将要发送给OC的数据保存在全局变量中，然后修改iframe的src来加载特定URL：``https://__wvjb_queue_message__``。

#### OC接收调用

因为webView要加载URL，同样先回调了webView的 ``shouldStartLoadWithRequest`` 方法。方法中识别了这是一个由WebViewJavascriptBridge发起的web端交互请求，进行拦截处理：

```objectivec
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
	...
    if ([_base isQueueMessageURL:url]) {
        NSString *messageQueueString = [self _evaluateJavascript:[_base webViewJavascriptFetchQueyCommand]];
        [_base flushMessageQueue:messageQueueString];
    }
	...
}
```

其中 ``webViewJavascriptFetchQueyCommand`` 的作用的获取之前保存在JS端的需要传递给OC的数据：

```objectivec
- (NSString *)webViewJavascriptFetchQueyCommand {
    return @"WebViewJavascriptBridge._fetchQueue();";
}
```

 通过JS端bridge对象的 ``_fetchQueue`` 获取：

```javascript
function _fetchQueue() {
    var messageQueueString = JSON.stringify(sendMessageQueue);
    sendMessageQueue = [];
    return messageQueueString;
}
```

可以看到传过来的数据是一个被序列化的数组字符串。

OC端拿到数据后，调用 ``flushMessageQueue`` 执行相应的逻辑。

```objectivec
- (void)flushMessageQueue:(NSString *)messageQueueString{
    if (messageQueueString == nil || messageQueueString.length == 0) {
        NSLog(@"WebViewJavascriptBridge: WARNING: ObjC got nil while fetching the message queue JSON from webview. This can happen if the WebViewJavascriptBridge JS is not currently present in the webview, e.g if the webview just loaded a new page.");
        return;
    }

    //反序列化json
    id messages = [self _deserializeMessageJSON:messageQueueString];
    for (WVJBMessage* message in messages) {
        if (![message isKindOfClass:[WVJBMessage class]]) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Invalid %@ received: %@", [message class], message);
            continue;
        }
        [self _log:@"RCVD" json:message];
        NSString* responseId = message[@"responseId"];
        //若传过来的数据中包含responseId字段，说明这是一个JS回调调用
        if (responseId)
        {
            WVJBResponseCallback responseCallback = _responseCallbacks[responseId];
            responseCallback(message[@"responseData"]);
            [self.responseCallbacks removeObjectForKey:responseId];
        }
        //若数据中不含responseId，说明这是一个JS发起的调用
        else
        {
            WVJBResponseCallback responseCallback = NULL;
            //若数据中含callbackId，说明JS端需要回调
            NSString* callbackId = message[@"callbackId"];
            if (callbackId) {
                //创建回调JS的Block
                responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    //向JS发送特定格式的数据来执行回调
                    [self _queueMessage:msg];
                };
            } else {
                responseCallback = ^(id ignoreResponseData) {
                    // Do nothing
                };
            }
            
            //根据handlerName获取messageHandlers中保存的方法
            WVJBHandler handler = self.messageHandlers[message[@"handlerName"]];
            
            if (!handler) {
                NSLog(@"WVJBNoHandlerException, No handler for message from JS: %@", message);
                continue;
            }
            
            //执行OC方法
            handler(message[@"data"], responseCallback);
        }
    }
}
```

处理逻辑和JS接收数据后的处理逻辑几乎一致，同样是根据传过来的数据中是否包含responseId字段来判断这是一个JS回调调用还是由JS发起的调用。若是JS回调调用，则从responseCallbacks集合中根据responseId取出回调block并调用；若是JS发起的调用，则最终从messageHandlers中根据handlerName取出对应的block调用。

#### JS回调

OC端在处理JS的调用时，若识别出这是一个JS发起的调用，而非回调调用，则根据传过来的数据中是否包含callbackId字段来决定是否需要回调JS。若需要，则创建回调block，并将该回调block传入OC方法，由OC业务层决定何时进行回调，以及回调什么样的数据。而该回调block的内容，无可厚非就是向JS发送特定格式的数据：

```objectivec
responseCallback = ^(id responseData) {
                    if (responseData == nil) {
                        responseData = [NSNull null];
                    }
                    WVJBMessage* msg = @{ @"responseId":callbackId, @"responseData":responseData };
                    //向JS发送特定格式的数据来执行回调
                    [self _queueMessage:msg];
                };
```

``_queueMessage`` 方法正是之前介绍的OC调用JS的方法。另外，JS在接收到回调数据后的处理也在之前介绍过了，不再赘述。

## 总结

- 交互前需要先对OC环境和JS环境进行初始化，JS环境的初始化通过Web页面加载时发送特定的URL来完成。
- WebViewJavascriptBridge在OC端和JS端各自维护一个bridge对象来保存开放给另一端的方法，以及自身调用另一端后的回调方法。前者通过handlerName来映射，后者通过callBackId标识唯一性。方法调用时必定携带handlerName，若需要回调，还需携带callBackId。
- WebViewJavascriptBridge中OC调用JS采用的是WebView提供的JS执行方法；而JS调用OC采用的是URL拦截的方式，OC端通过识别特定的URL来区分是否需要拦截，并做相应的逻辑处理。