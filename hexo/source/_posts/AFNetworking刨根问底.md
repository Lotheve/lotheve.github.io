---
title: AFNetworking刨根问底
date: 2018-05-21 23:53:48
tags:
---

![](http://oubmw34rc.bkt.clouddn.com/blog/AF/afnetworking-logo.png)

AFNetworking网络框架在iOS开发中的霸主地位已经根深蒂固，本篇将基于3.2.1版本对框架的几个核心模块做一波分析。首先对于框架整体的架构，简单归纳如下：

![](http://oubmw34rc.bkt.clouddn.com/blog/AF/afnetworking1.png)

## AFURLSessionManager

#### 模块概要

AFURLSessionManager是AF最核心的模块，用来创建请求的task实体。每个manager都维护了一个NSURLSession对象及其配置，用来创建task。对外开放了若干创建task对象的接口（包括DataTask、DownloadTask、UploadTask）以及设置回调方法的接口，整体功能并不复杂。关于该模块的内容，我决定以问答的形式进行展开，读者可以在自行阅读源码的基础上，结合这几个问题的分析来理解。

####  session和task是一对多的关系，如何将task的回调方法保存下来？

我们知道，一个session可以创建多个task，一个sessionManager维护一个session，因此sessionManager和task是一对多的关系。sessionManager类提供了创建task的接口，接口支持定制task的上/下行进度、请求落地的回调，例如：

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler;
```

既然是一对多的关系，sessionManager内部必然要将task的回调方法保存起来，以便在task请求的过程中调用，从而能够实现定制每个task的回调。那么内部是怎么实现的呢？

AFURLSessionManager的实现文件主要包含了两个类：AFURLSessionManagerTaskDelegate 和 AFURLSessionManager。AFURLSessionManager每创建一个task，就会创建一个对应的AFURLSessionManagerTaskDelegate对象，并将两者关系对应起来维护在一个字典中。这两个类都履行了作为请求回调代理对象的职责，只不过，AFURLSessionManager是session级别的，只要是当前session下的task，都会回调AFURLSessionManager中的代理方法，如果只是这样，回调就不能细化到task级别，这就很鸡肋了。毕竟在实际业务中，不同的请求的回调处理逻辑大相径庭。而AFURLSessionManagerTaskDelegate的存在则是为了定制task级别的回调，主要涉及4个请求过程相关的回调：上行进度、下行进度、请求落地以及文件下载完成，这些回调block在每次创建task的时候都由上层传进来（不传默认为nil），然后创建一个专门的delegate对象来持有这个task的回调方法，因为task和delegate对象是一一对应的，因此这个task的回调方式就被保存下来了。

以下行进度为例，任意一个请求，当请求接收到数据，AFURLSessionManager的 `URLSession: dataTask: didReceiveData:` 都会收到回调通知，不过这个方法里只能处理session级别的回调逻辑，要对应到task级别，操作就是找到task对应的AFURLSessionManagerTaskDelegate对象，将消息转发给该对象的 `URLSession: dataTask: didReceiveData:` 方法。因为delegate对象存有对应task的下行回调方法，这样一来，就能够将下行进度的回调细化到task级别了。下面是AFURLSessionManager接收请求数据的代理方法：

```objc
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    // 取出task对应的delegate，将消息转发给delegate对象，从而将回调细化到task级别
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    [delegate URLSession:session dataTask:dataTask didReceiveData:data];
    // session级别的回调逻辑
    if (self.dataTaskDidReceiveData) {
        self.dataTaskDidReceiveData(session, dataTask, data);
    }
}
```

所以AFURLSessionManagerTaskDelegate的存在，就是为了上层能够定制task主要的几个回调。

#### 涉及的线程？

1. **请求在哪个线程发起？**

   这个问题无法回答具体在哪个线程发起。每创建一个线程，都会分配一块固定大小的栈内存（512K），另外线程的切换也有开销，因此发请求这一操作是由系统根据当前cpu及内存的状态，决定在哪个线程执行，以及是否需要创建新的线程来执行的。

2. **请求的内部回调在哪里执行？**

   指的是AFURLSessionManager中的NSURLSessionDelegate 、NSURLSessionTaskDelegate 等一系列协议中的回调方法在哪个线程执行。这些回调的执行位置取决于创建session的时候，指定的delegateQueue：

   ```objective-c
   self.operationQueue = [[NSOperationQueue alloc] init];
   self.operationQueue.maxConcurrentOperationCount = 1;
   self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
   ```

   官方对于delegateQueue参数的说明如下：

   > An operation queue for scheduling the delegate calls and completion handlers. The queue should be a serial queue, in order to ensure the correct ordering of callbacks. If nil, the session creates a serial operation queue for performing all delegate method calls and completion handler calls.

   根据描述，delegateQueue必须是一个串行队列，目的是为了使回调方法顺序执行。如何理解这个顺序执行？例如 `URLSession: dataTask: didReceiveData:` 方法，一个请求过程可能会回调多次，必须得保证这多个回调是时序的，否则进度回调就会有问题。如若使用并发队列，就无法保证时序了。AF里使用了一个自定义队列，并设置最大并发数为1，这就相当于是一个串行队列了。因此，请求的回调是在非主线程的线程中顺序执行的。若是将operationQueue设置为 `[NSOperationQueue mainQueue] ` ，那就是在主线程中执行了。

3. **请求对外回调在哪里执行？**

   首先要说的是，对外回调，主要涉及4个回调，申明在AFURLSessionManagerTaskDelegate 中：

   ```objective-c
   @property (nonatomic, copy) AFURLSessionDownloadTaskDidFinishDownloadingBlock downloadTaskDidFinishDownloading;
   @property (nonatomic, copy) AFURLSessionTaskProgressBlock uploadProgressBlock;
   @property (nonatomic, copy) AFURLSessionTaskProgressBlock downloadProgressBlock;
   @property (nonatomic, copy) AFURLSessionTaskCompletionHandler completionHandler;
   ```

   这里需要区分一下请求落地的回调和另外几个。在AFURLSessionManager的头文件中，可以看到开放了两个相关属性：completionQueue、completionGroup ，前者是用来指定回调队列的，后者用来指定关联的group，不过这两个属性是给请求落地的回调（也就是completionHandler）用的。代码如下（仅保留了主要代码）：

   ```objective-c
   // AFURLSessionManagerTaskDelegate.m
   - (void)URLSession:(__unused NSURLSession *)session
                 task:(NSURLSessionTask *)task
   didCompleteWithError:(NSError *)error
   {
    	...
       if (error) {
           userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;
           //若上层没有指定回调队列，则默认在主线程回调 
           //若上层没有指定回调group，则默认使用内部的group
           dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
               if (self.completionHandler) {
                   self.completionHandler(task.response, responseObject, error);
               }
   		   //主线程发送通知
               dispatch_async(dispatch_get_main_queue(), ^{
                   [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
               });
           });
       } else {
           //异步并发调用 进行解析数据和对外回调
           dispatch_async(url_session_manager_processing_queue(), ^{
               NSError *serializationError = nil;
               responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];
               if (self.downloadFileURL) {
                   responseObject = self.downloadFileURL;
               }
               if (responseObject) {
                   userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
               }
               if (serializationError) {
                   userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
               }
   		   //回调方式同上
               dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                   if (self.completionHandler) {
                       self.completionHandler(task.response, responseObject, serializationError);
                   }
                   dispatch_async(dispatch_get_main_queue(), ^{
                       [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                   });
               });
           });
       }
   }
   ```

   因此结论是若上层没有指定回调队列，**请求落地**的回调默认在主线程执行。

   而对于**上行进度**（uploadProgressBlock）和**下行进度**（downloadProgressBlock）的回调，则是另外的逻辑。首先要搞清楚上下行进度回调是怎么做到的？不清楚的童鞋请自行跳到下个Topic先。

   现在假设你已经知道进度回调是借助KVO实现的了，进度回调的方法在KVO的代理方法中执行。那么问题就是KVO的代理方法在哪个线程执行了。答案是：**监听的属性在哪个线程修改，就在哪个线程通知回调**。很明显，属性是间接在请求的代理方法中修改的。之前讲过，代理方法是在非主线程的线程中执行的，因此，进度的回调，同样也是在非主线程执行的。实际业务中，可能经常会遇到监听上传请求进度更新进度条的UI操作，这时候就需要注意要在主线程执行这一操作了。*PS不解为何进度回调不设置成跟请求落地的回调一样，默认在主线程回调？*

   另外还有个下载完成的回调方法，也是间接在代理方法中调用的（读者可自行查阅代码），因此也在非主线程执行。

####上下行进度回调的实现

AF中使用NSProgress表示请求的进度，AFURLSessionManagerTaskDelegate对象持有两个分别表示上行进度和下行进度的NSProgress对象。

```objective-c
@property (nonatomic, strong) NSProgress *uploadProgress;
@property (nonatomic, strong) NSProgress *downloadProgress;
```

当回调进度相关的代理方法时，NSProgress对象的进度会发生更新。以上行进度为例：

```objective-c
//AFURLSessionManager.m
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend
{
	...
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];
    if (delegate) {
        [delegate URLSession:session task:task didSendBodyData:bytesSent totalBytesSent:totalBytesSent totalBytesExpectedToSend:totalBytesExpectedToSend];
    }
	...
}
```

可以看到回调被转发给了task对应的delegate对象：

```objective-c
//AFURLSessionManagerTaskDelegate.m
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend{
    self.uploadProgress.totalUnitCount = task.countOfBytesExpectedToSend;
    self.uploadProgress.completedUnitCount = task.countOfBytesSent;
}
```

delegate对象做的事仅仅是更新了uploadProgress的totalUnitCount和completedUnitCount，这怎么就触发了上行进度回调方法的执行呢？相信你已经猜到了，没错就是KVO。delegate对象在创建的时候就注册了对NSProgress对象的fractionCompleted属性的监听:

```objective-c
[progress addObserver:self
           forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
      		 options:NSKeyValueObservingOptionNew
              context:NULL];
```

当completedUnitCount更新时，fractionCompleted在内部也会被更新，从而触发了KVO的代理方法。

```objective-c
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context {
   if ([object isEqual:self.downloadProgress]) {
        if (self.downloadProgressBlock) {
            self.downloadProgressBlock(object);
        }
    }
    else if ([object isEqual:self.uploadProgress]) {
        if (self.uploadProgressBlock) {
            self.uploadProgressBlock(object);
        }
    }
}
```

可见是在KVO的代理方法中执行进度回调方法的。

## AFHTTPSessionManager

AFHTTPSessionManager是AFURLSessionManager的子类，他的职能就是封装了包括GET、POST、PUT、HEAD等各种方式的请求接口，方便上层调用。其内部实现很简单，各个不同请求方式的接口，最终都调到同一个方法：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                  uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgress
                                downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    // 创建HTTP请求对象
    NSError *serializationError = nil;
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }
    // 调用下层的请求接口创建task对象
    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request
                          uploadProgress:uploadProgress
                        downloadProgress:downloadProgress
                       completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];
    return dataTask;
}
```

该方法就做了两件事：

1. 创建HTTP请求对象：使用requestSerializer创建请求对象，设置好请求方法，URL以及请求参数。requestSerializer是AFHTTPSessionManager的一个属性，用来创建HTTP请求对象（NSURLRequest），可以由上层指定类型，默认使用AFHTTPRequestSerializer，具体将在下节详细讲。另外还有个responseSerializer，是用来解析响应数据的，不过它是由AFURLSessionManager持有的，在AFURLSessionManagerTaskDelegate的处理请求完成的方法中可以见到它的身影。
2. 调用下层的请求接口创建task对象：通过创建的request对象，调用下层的接口创建task对象然后返回。

事实上也不是所有HTTP请求都会调到这个方法，唯一有区别的就是支持文件上传的的POST请求，因为其创建请求对象的方式有所不同，以及调用的下层接口也有所不同，所以单独拎出来处理。具体可以看这个方法的实现：

```objective-c
- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
     constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                      progress:(nullable void (^)(NSProgress * _Nonnull))uploadProgress
                       success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure;
```

## AFURLRequestSerialization

#### 模块概要

请求序列化器，上一节提到过这个类，AFHTTPSessionManager持有它，用来创建HTTP请求对象，因此笔者习惯称之为请求生成器。该类的实现文件有1400多行代码，比AFURLSessionManager还要长，但是它干的事却可以用一两句话来概括：**用于创建完备的HTTP请求对象（NSURLRequest），包括设置好请求方式、请求头域、请求体，以及请求的一些配置，例如是否使用默认的cookie方案、是否支持蜂窝移动等。**而之所以代码显得略长，主要是因为不同的表单数据编码类型的存在，需要一些复杂的操作，例如：表单数据由字典格式化为application/x-www-form-urlencoded格式的query串、multipart/form-data编码类型的表单请求的数据流格式化等。根据实现的差别主要分为三种请求对象：1.不含请求体的GET、HEAD、Delete请求；2.application/x-www-form-urlencoded编码类型的表单请求；3.multipart/form-data编码类型的表单请求。后面将对这3种类型的请求对象的创建流程进行讲解， 一些次要的辅助实现读者可自行阅读。

>对于不同表单编码类型的区别，参考 [[四种常见的 POST 提交数据方式]](https://imququ.com/post/four-ways-to-post-data-in-http.html)

这一模块主要涉及1个协议：AFURLRequestSerialization 和3个类：AFHTTPRequestSerializer 、AFJSONRequestSerializer、AFPropertyListRequestSerializer。关系如下：

![](http://oubmw34rc.bkt.clouddn.com/blog/AF/afnetworking2.png)

+ *AFURLRequestSerialization*：协议，包含一个根据字典参数创建不同MIME类型请求的方法的申明。
+ AFHTTPRequestSerializer：默认的请求生成器，遵循AFURLRequestSerialization，提供默认的请求生成逻辑，包括创建不含请求体的GET、HEAD、DELETE请求 ，以及application/x-www-form-urlencoded编码类型的表单请求。
+ AFJSONRequestSerializer：AFHTTPRequestSerializer的子类，当表单数据需要使用application/json类型编码时使用。
+ AFPropertyListRequestSerializer：AFHTTPRequestSerializer的子类，当表单数据需要使用application/x-plist类型编码时使用。

#### 不含请求体的GET、HEAD、Delete请求

我在阅读三方库源码的时候，一般都会先浏览一下头文件，大致拿捏一下文件结构以及各个类、方法的职责，以便找到一个合适的切入点。对于AFURLRequestSerialization来说，我的切入点就是`requestBySerializingRequest: withParameters:error:` 方法，因为AFURLRequestSerialization的职能就是创建请求对象，这个方法闻其名正中下怀，并且在AFHTTPSessionManager中就是通过这个方法创建请求对象的，可见该方法也是与外层交互的接口。

```objc
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    NSURL *url = [NSURL URLWithString:URLString];
	//设置请求头
    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;
	//设置请求配置
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
```

最直观的就是设置了请求方式及请求对象的一些配置。AFURLRequestSerialization开放了几个配置请求的属性，例如allowsCellularAccess（是否允许使用蜂窝移动 ) 、缓存策略（cachePolicy ）等，如果外层配置了相关属性，内部会记录该属性名到mutableObservedChangedKeyPaths数组中，创建请求对象的时候，根据该数组得知哪些属性配置过了，从而配置到请求中。

再者就是调用了 `requestBySerializingRequest:withParameters:` 方法，这就是那个协议申明的方法。来看看它在AFHTTPRequestSerializer中的实现：

```objc
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);
    NSMutableURLRequest *mutableRequest = [request mutableCopy];
	//设置请求头域
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];
    //请求参数格式化成application/x-www-form-urlencoded类型要求的格式                                   
    NSString *query = nil;
    if (parameters) {
        if (self.queryStringSerialization) {
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);
            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }
                return nil;
            }
        } else {
            //key1=value1&key2=value2&... 会进行URL编码
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }
    //对于GET, HEAD, DELETE请求，将query参数拼接在URL中
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
        if (!query) {
            query = @"";
        }
        //其他方式的请求，需要把格式化后的表单数据设置到请求体中，并设置请求头域中的内容类型为application/x-www-form-urlencoded
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }
    return mutableRequest;
}
```

首先，设置了请求头域。HTTPRequestHeaders数组中包含了两个默认的头域：Accept-Language 、User-Agent。接下来就是设置表单数据了，因为对于不含文件的表单请求，默认是使用application/x-www-form-urlencoded类型编码的，因此要先将表单数据格式化成以&分割的键值对形式：key1=value1&key2=value2&...。之后要做的是将格式化后的表单数据设置到请求对象中。对于GET、HEAD、DELETE方式的请求，只要将表单数据拼接在URL上即可。

至此，一个完备的GET/HEAD/DELETE请求对象就创建好了，返回给外层使用即可。

#### application/x-www-form-urlencoded编码类型的表单请求

继续看上面的 `requestBySerializingRequest:withParameters:` 方法，对于非GET/HEAD/DELETE方式的请求，操作是将表单数据编码后设置到请求体中的，并设置内容类型为application/x-www-form-urlencoded类型。

需要注意的是：此处的表单数据是application/x-www-form-urlencoded类型的（key1=value1&key2=value2&...），所以指定"content-type"为application/x-www-form-urlencoded。如果是jSON格式的表单数据，即HTTPBody是参数字典通过NSJSONSerialization序列化得到的JSON数据，那么表单数据格式为JSON格式，则需要将"content-type"指定为application/json。AFJSONRequestSerializer就是做这个事的，作为AFHTTPRequestSerializer的子类，唯一的区别就是表单数据的编码类型不同，即"Content-Type"指定的类型不同；另外还有个AFPropertyListRequestSerializer，也是AFHTTPRequestSerializer的子类，当表单数据编码类型为application/x-plist的时候使用。

#### multipart/form-data编码类型的表单请求

除了支持上面两种类型的请求，还有一种比较特殊的表单请求，需要另外处理。这就是 multipart/form-data编码类型的表单请求，可以用来上传文件等数据流。在将AFHTTPSessionManager的时候提到过有一个比较特殊的支持文件上传的POST请求，其创建请求对象的方式和其他请求有所区别：

```objc
- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
     constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                      progress:(nullable void (^)(NSProgress * _Nonnull))uploadProgress
                       success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
	...
    NSError *serializationError = nil;
    NSMutableURLRequest *request = [self.requestSerializer multipartFormRequestWithMethod:@"POST" URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters constructingBodyWithBlock:block error:&serializationError];
	...
}
```

可以看到是通过 ``multipartFormRequestWithMethod:URLString:parameters:constructingBodyWithBlock:`` 方法创建的请求对象。从方法名可知该方法就是用来创建multipart/form-data编码类型的表单请求的。

>关于multipart/form-data，不了解的童鞋可以先参考[Multipart/form-data](https://www.jianshu.com/p/e810d1799384)

```objc
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(NSDictionary *)parameters
                              constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError *__autoreleasing *)error
{
    NSMutableURLRequest *mutableRequest = [self requestWithMethod:method URLString:URLString parameters:nil error:error];
    __block AFStreamingMultipartFormData *formData = [[AFStreamingMultipartFormData alloc] initWithURLRequest:mutableRequest stringEncoding:NSUTF8StringEncoding];
    // 拼接非文件类型表单数据流
    if (parameters) {
        for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
            NSData *data = nil;
            if ([pair.value isKindOfClass:[NSData class]]) {
                data = pair.value;
            } else if ([pair.value isEqual:[NSNull null]]) {
                data = [NSData data];
            } else {
                data = [[pair.value description] dataUsingEncoding:self.stringEncoding];
            }

            if (data) {
                [formData appendPartWithFormData:data name:[pair.field description]];
            }
        }
    }
    // 拼接其他数据流，这个block是上层传入的，上层可以通过此block注入要上传的文件数据流或任何其他表单数据
    if (block) {
        block(formData);
    }
    // 数据流拼接完毕后进行整理，设置请求对象到HTTPBodyStream中，设置"Content-Type"为"multipart/form-data",并计算数据长度，设置到"Content-Length"头域中
    return [formData requestByFinalizingMultipartFormData];
}
```

关于multipart/form-data类型的表单处理，因为其数据格式较为复杂，而且要支持不同类型表单项的添加方法，因此相关源码占了较大的篇幅。不过主要的处理逻辑很简单，只是实现较为复杂。AFStreamingMultipartFormData这个类主要是用来封装表单数据流拼接相关操作的，首先是对字典项参数中的数据一一调用了 `appendPartWithFormData` 方法进行拼接，而后执行了一个外部传入的block，用来拼接外部的表单项，可能是文件、图片，也可能是简单的key-value键值对。最后，数据流拼接完毕后，再进行整理：

```objc
- (NSMutableURLRequest *)requestByFinalizingMultipartFormData {
    if ([self.bodyStream isEmpty]) {
        return self.request;
    }
    [self.bodyStream setInitialAndFinalBoundaries];
    [self.request setHTTPBodyStream:self.bodyStream];
    [self.request setValue:[NSString stringWithFormat:@"multipart/form-data; boundary=%@", self.boundary] forHTTPHeaderField:@"Content-Type"];
    [self.request setValue:[NSString stringWithFormat:@"%llu", [self.bodyStream contentLength]] forHTTPHeaderField:@"Content-Length"];
    return self.request;
}
```

设置数据流的上下边界，这样表单数据流就完全成型了，设置到HTTPBodyStream中。最后再设置请求头域中的内容类型以及内容宽度。至此，一个multipart/form-data表单类型的请求才算创建完毕，可以返回给外层使用了。

#### 自定义请求

AF仅提供了各种表单类型格式、JSON格式、以及plist格式的数据的上传，如果有需要可以自定义创建一个指定数据格式的请求序列化器，例如XML格式。相关操作只需要继承AFHTTPRequestSerializer，然后实现AFURLRequestSerialization 协议中申明的方法可以，具体可以参照AFJSONRequestSerializer。

## AFURLResponseSerialization

#### 模块概要

响应序列化器，用来解析响应数据，由AFURLSessionManager持有，习惯称之为响应解析器。该模块代码结构直观上比AFURLRequestSerialization清晰很多，主要包含了几种不同类型的响应数据解析方式，包含：JSON、XML、Plist、image。另外还有一个协议，申明了一个用来执行数据解析的方法。类结构如下：

![](http://oubmw34rc.bkt.clouddn.com/blog/AF/afnetworking3.png)

+ AFURLResponseSerialization：协议，申明了解析响应数据的方法
+ AFHTTPResponseSerializer：响应解析器的基类，仅提供了校验响应MIME类型及状态码的方法，不进行数据解析
+ AFJSONResponseSerializer ：JSON解析器
+ AFXMLParserResponseSerializer ：XML解析器
+ AFPropertyListResponseSerializer ：Plist解析器
+ AFImageResponseSerializer ：Image解析器
+ AFCompoundResponseSerializer ：组合解析器，使用包含的多个解析器去碰撞解析，解析成功即可。

下面以JSON类型数据解析为例，讲解一下解析的流程是怎么样的。

##### AFJSONResponseSerializer 

在JSON解析器初始化方法中，指定了几种允许的MIME类型：

```objc
self.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", nil];
```

这是一个集合属性，用来指定期望接收的MIME类型，如果响应的数据类型不在集合范围内，会被认定为这是一个出错的请求。另外还有一个acceptableStatusCodes属性，用来指定期望的响应状态码，即若实际响应的状态码不在期望的范围内，同样会被认定为是一个出错的请求，错误信息会经过包装后传递给外层。默认期望的成功请求状态码为200-299，是在基类AFHTTPResponseSerializer中指定的。

外层使用时是通过调用 `responseObjectForResponse: data: error:` 来进行数据解析的，该方法就是协议AFURLResponseSerialization中申明的方法，因此若想自定义一种解析器，继承AFHTTPResponseSerializer ，实现该方法即可。AFJSONResponseSerializer中该方法的实现如下：

```objc
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    //校验响应的内容类型及状态码 对于不符合期望的MIME类型的响应直接ruturn nil
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }
    //JSON解析
    BOOL isSpace = [data isEqualToData:[NSData dataWithBytes:" " length:1]];
    if (data.length == 0 || isSpace) {
        return nil;
    }
    NSError *serializationError = nil;
    id responseObject = [NSJSONSerialization JSONObjectWithData:data options:self.readingOptions error:&serializationError];
    if (!responseObject)
    {
        if (error) {
            *error = AFErrorWithUnderlyingError(serializationError, *error);
        }
        return nil;
    }
    //是否移除JSON对象中值为空的项
    if (self.removesKeysWithNullValues) {
        return AFJSONObjectByRemovingKeysWithNullValues(responseObject, self.readingOptions);
    }
    return responseObject;
}
```

1. 检查响应数据是否符合期望的MIME类型以及状态码
2. 进行JSON解析
3. 根据removesKeysWithNullValues属性决定是否移除JSON对象中值为空的项 
4. 返回解析后的JSON对象

AFURLSessionManager中使用解析器的地方是在NSURLSessionTaskDelegate的 `URLSession: task: didCompleteWithError:` 方法中，即请求落地后调用解析器的 `responseObjectForResponse: data: error:` 方法进行响应数据的解析，而后回调给上层。

## AFSecurityPolicy

苹果一再强调HTTPs，虽然时至今日（2018.5）还是没有强制使用HTTPs，但是相信大多数APP都已经接入HTTPs了。AFSecurityPolicy是用来验证HTTPs证书的工具类，支持对CA证书及自签证书的验证方式进行配置。

主要包含3中验证策略：

+ AFSSLPinningModeNone：表示不做SSL pinning，即不用证书绑定的方式验证，只跟浏览器一样在系统的信任机构列表里验证服务端返回的证书。若证书是CA签发的就会通过，若是自签证书是无法通过的。 
+ AFSSLPinningModePublicKey：用证书绑定方式验证，客户端要有服务端的证书拷贝，只是验证时只验证证书里的公钥，不验证证书的有效期等信息。 
+ AFSSLPinningModeCertificate：也是用证书绑定方式验证证书，需要客户端保存有服务端的证书拷贝，这里验证分两步，第一步验证证书的有效性（包括签名/格式/域名/有效期等信息），第二步是对比本地证书与服务器证书是否一致。 

AF中默认的验证策略是AFSSLPinningModeNone，对于这种策略网上很多资料都解释成不进行任何校验，这种说法完全就是误导人，它表示的只是不使用证书绑定的方式校验证书，因此这是一种当使用CA证书而非自签证书的时候使用的一种策略。

AFURLSessionManager中，是在请求的鉴权代理方法中进行证书验证的，具体有两处相关的回调的代理，分别是 `URLSession: didReceiveChallenge: completionHandler:` 和 `URLSession: task: didReceiveChallenge: completionHandler:` 。关于两者的区别，前者只在 challenge.protectionSpace.authenticationMethod 为下面4个中的一个时才会调用：

- NSURLAuthenticationMethodNTLM
- NSURLAuthenticationMethodNegotiate
- NSURLAuthenticationMethodClientCertificate
- NSURLAuthenticationMethodServerTrust

若前者未实现，则调用后者。

```objc
if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
    if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
        credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
        if (credential) {
            disposition = NSURLSessionAuthChallengeUseCredential;
        } else {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    } else {
        disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
    }
} else {
    disposition = NSURLSessionAuthChallengePerformDefaultHandling;
}
```

challenge表示一个认证的挑战对象，内含此次认证的所有信息。其属性protectionSpace保存了需要认证的保护空间，每个protectionSpace都保存了认证主机地址、端口、认证方法等重要信息。当认证方法为NSURLAuthenticationMethodServerTrust时，就会调用securityPolicy的方法进行认证。最后根据认证结果回调不同的disposition和disposition参数。

下面主要看核心的认证方法 `evaluateServerTrust: forDomain:` ，方法主要传入服务器认证对象及请求域名 :

```objc
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{
    //1、不能隐式地信任自签证书？个人理解，对于不使用证书绑定的方式验证或者不提供绑定证书的情况，若设置了允许无效证书，即allowInvalidCertificates为YES，同时又要求验证域名，这种设置是不安全的的，直接返回NO。
    //此处不是很理解，望各路大神赐教！！
    if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
        NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
        return NO;
    }
    //2、设置校验策略。若需要校验域名，则根据域名创建一个校验策略；否则创建一个X509标准的校验策略，用来校验证书的有效性（不会核对域名）
    NSMutableArray *policies = [NSMutableArray array];
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }
    SecTrustSetPolicies(serverTrust, (__bridge CFArrayRef)policies);
    //3、验证证书有效性
    //3.1、不使用绑定证书校验的情况，若允许无效证书则直接返回YES，否则返回校验CA证书的验证结果。
    //3.2、使用绑定证书校验的情况，若证书无效，并且不允许无效证书，则直接返回NO。
    if (self.SSLPinningMode == AFSSLPinningModeNone) {
        return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
    } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
        return NO;
    }
    //4、根据SSLPinningMode对服务器证书做校验
    switch (self.SSLPinningMode) {
        //4.1、CA校验，代码不会执行到此处
        case AFSSLPinningModeNone:
        default:
            return NO;
        //4.2、绑定证书校验
        case AFSSLPinningModeCertificate: {
            NSMutableArray *pinnedCertificates = [NSMutableArray array];
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);
            //4.2.1、先校验证书的有效性
            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }
            //4.2.2、匹配本地证书与服务器证书，若本地有能够匹配上的证书，则校验通过
            NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    return YES;
                }
            }
            return NO;
        }
        //4.3、公钥校验，若本地证书中，有公钥是能够和服务器证书的公钥匹配的，则验证成功。
        case AFSSLPinningModePublicKey: {
            NSUInteger trustedPublicKeyCount = 0;
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);
            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            return trustedPublicKeyCount > 0;
        }
    }
    return NO;
}
```

验证过程已经在代码中注解了，另外一个内部的方法实现例如获取证书中的公钥，有兴趣的读者可以自行阅读。此处仅根据个人理解做几点说明：

1. **使用绑定证书校验，即校验自签证书时，为什么要先校验证书的有效性，再匹配本地证书与服务器证书是否完全一致？**

   个人理解，证书校验一步校验的是证书的域名、有效期、证书格式、有效性（验签）等，但是即便本地不含与服务端一致的证书，只要有相同公钥的证书，就能使验签通过，另外域名、有效期等的校验，则与本地证书完全无关。因此要想完整地验证自签证书，就得先校验证书的域名、有效期、有效性等信息，然后再匹配本地是否有该证书，这样的验证逻辑才是最完备的。若没有前面一步，那证书即便过期了也能永久使用。

2. **公钥验证的使用场景？**

   所谓公钥校验就是用本地证书中的公钥匹配服务器证书的公钥，若有能够匹配的公钥，则验证成功。这种验证方式是一种极简易的验证方式，好处是即便服务器证书过期了，客户端也无需更换证书，只要服务器在创建证书时依旧使用老证书的密钥对即可。

3. **是否有必要校验域名？**

   答案是能校验则校验。校验域名能够有效避免中间人攻击，若不校验域名，中间人劫持了HTTPs请求，并做了转发，这时候如果不校验域名，只要中间人给客户端的是一张合法的CA颁发的证书，客户端将毫不知情。这种情况下若进行证书校验，客户端就会因为服务器证书（实际上是中间人给的证书）中的域名匹配不上而拒绝连接。不过，如果你的请求是某个域名的子域名，而使用根域名的证书，例如证书上的域名为google.com，而请求是mail.google.com，此时若校验域名是不通过的，因此只能忽略域名的校验。当然不差钱的话，含通配符域名的证书你值得拥有。

4. **自签证书是否安全？**

   使用自签证书时，只要本地保存一份与服务器一致的证书即可，那么这种操作安全吗？答案是肯定的，所谓安全与否，主要是看能否避开中间人攻击。因为中间人无法拿到服务器私钥，因此是无法建立与客户端的TLS连接的（TLS连接过程中加密密钥的约定需要服务端的私钥）。

## 结语

AFNetworking作为当前iOS开发中使用最为广泛的基于OC的网络库，基本是三方库中不可或缺的一员。整个库的架构很清晰，外围还有用来检测网络的Reachability模块，以及用来扩展各个组件网络功能的UIKit模块，因其都属于即插即用的模块，并非AF的核心，本篇不再对其进行讲解。实际项目中的网络框架，大多根据实际业务在AF的基础上进行二次封装，常用的功能像批量请求、单一请求的取消、文件上传等，可能都需要更为精简的封装，以降低业务层的使用门槛。