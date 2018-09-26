---
title: iOS热修复技术及JSPatch源码探索
date: 2017-03-20 14:48:16
tags:
---

## 热修复简介

对于iOS应用而言，目前app store的审核周期可能通常维持在1-2个星期。倘若一个线上的应用出现了一些bug，甚至是致命的崩溃，这时候假如按照苹果的套路乖乖重新发布一个版本，然后静静等待看似漫无期限的审核周期，最终结果就是：用户大量流失。因此，对于一些线上的bug，需要有及时修复的能力，这就是所谓的热修复（hotfix）。

由于苹果应用审核周期长（毕竟苹果尿性高），hotfix就是一个非常重要的角色。相比而言，Android的审核可能通常在1天之内就能完成，显得就不是那么重要了。

与热修复有点划不清界限的另一个词叫热更新，两者的手段是一样的，都是通过下发补丁文件来修改线上工程的运作，只不过热修复的目的是修复线上bug，而热更新的目的则是对线上产品进行临时的更新。

#### 苹果的态度
《iOS Developer Program Information》中 *Sample Apple Developer Program Requirements - 3.3.2* 条例规定：

>3.3.2 Except as set forth in the next paragraph, an Application may not download or install executable code. Interpreted code may only be used in an Application if all scripts, code and interpreters are packaged in the Application and not downloaded. The only exceptions to the foregoing are scripts and code downloaded and run by Apple's built-in WebKit framework or JavascriptCore, provided that such scripts and code do not change the primary purpose of the Application by providing features or functionality that are inconsistent with the intended and advertised purpose of the Application as submitted to the App Store. 


大致意思是，苹果可能不允许动态下发可执行代码或者脚本文件，但通过苹果JavaScriptCore.framework或WebKit执行的代码除外。

## iOS热修复方案

#### WebView加载HTML5动态更新
这种方案只针对于嵌套H5页面的web app，其实是借助了web本身的热修复能力。只需将修复或者更新后的web文件部署到服务器，即可实时更新终端界面或逻辑的效果。这种方案局限性显而易见，只能对h5界面作热修复和更新，对原生模块无能为力。

应用中嵌入h5页面大多考虑这点：重效率轻交互，适合业务变动较为频繁，对用户体验要求不高的模块。由于是网页，只需外面套一个WebView容器即可跨平台复用，开发效率不言而喻；但是h5性能一直是个老生常谈的问题，交互体验和原生相比还是有一定差距的。

#### Dynamic Framework方案
静态库和动态库都是二进制文件，两者的区别在于，静态库在编译时就被链接进可执行文件，成为程序的一部分；而动态库并不被打包进程序，是在程序运行的时候被动态加载。前者相对于编译时，后者相对与运行时。

苹果在XCode6中开放了iOS的动态库，这样借助动态库可以做很多事，例如越狱开发中的tweak开发，本质就是借助动态库来hook系统函数或者app方法，达到修改系统功能或者app功能的目的。另外，通过动态下发动态库可以实现应用版本升级的目的，也就是热更新。如果应用中相关模块本身就是用动态库实现的，出现了bug，还可以通过动态下发新的动态库来实现热修复。

实际中，更多的是用来热更新。因为要求事先在应用中写好加载动态库以及调用相关方法的代码，说明事先就预见了当前环境的可扩展性，这更符合热更新的动机。

但是有个最大的问题就是，按照苹果的说明是不允许动态下发可执行代码的，即便下发了也可能无法加载，但是比较矛盾的是，苹果在iOS8开放了NSBundle挂载动态库的接口，所以到底是几个意思呢？事实上估计也很少有人实践过吧，因此通过下发动态库来实现热修复和热更新好像本身就是个迷，从技术上来讲必定行得通，能不能通过审核就不知道了。
[Apple Developer Forums：《Is it possible to submit an app to App store with a dynamic framework that has the simulator slice in it?》](https://forums.developer.apple.com/thread/21496) 
[知乎：《现在有线上iOS app 支持动态链接库 动态加载过审的例子了么？》](https://www.zhihu.com/question/34836695) 

#### 基于RN/Weex跨平台方案的动态更新
React Native和Weex均是用前端开发方式开发原生应用的框架，核心是JS与原生的交互。利用JS动态更新的性质，可以实现应用的热修复以及热更新。

事实上，这两套框架更多的是作为整套功能模块的开发方案在用，其主要特点有几点：

1. 跨平台 （RN支持iOS和安卓，Weex支持iOS、安卓、Web）
2. 面向前端 （RN采用的语法是前端React框架的语法，Weex采用的是前端vue框架的语法）
3. 可扩展 （支持扩展原生组件或者API）
4. 动态更新 （JS动态更新的能力）

只是其实现方式使其拥有了动态更新的能力，而非其出发点就是为了满足动态更新。因此利用RN和Weex来实现hotfix以及热更新的最大的局限性，就是只能针对使用了这套方案的模块，对原生模块同样无能为力。

[React Native官网](http://facebook.github.io/react-native/) 
[React Native源码](https://github.com/facebook/react-native) 
[Weex官网](https://alibaba.github.io/weex/) 
[Weex源码](https://github.com/alibaba/weex)


#### WaxPatch 基于lua控制动态更新

WaxPatch是一套用lua编写的iOS框架，用户可以使用lua来调用iOS SDK中的接口，并且借助runtime实现了对现成方法的替换，从而达到hotfix的目的。WaxPatch的初衷就是给iOS应用打补丁，但是这套框架已经年久失修，有很多缺点：

1. 作者已经停止维护，SDK中的很多方法不再支持，例如不支持block
2. 采用的是lua脚本语言，而iOS并没有内置lua引擎，因此要手动添加到工程，无疑会增加包的体积
3. 不支持arm64架构
4. 文档匮乏

基于WaxPatch的诸多弊端，其替代者JSPatch应运而生！

#### JSPatch 基于JS控制动态更新

区别于WaxPatch，JSPatch是用JavaScript实现的用以iOS hotfix的一套框架。JSPatch依赖于JavaScriptCore，但是由于JavaScriptCore是在iOS7中引进的，因此JSPatch只支持iOS7以上。另外，JS脚本完全符合苹果要求的下发规则。

[JSPatch源码](https://github.com/bang590/JSPatch)

## JSPatch

### 原理
JSPatch允许用JS调用原生方法、替换原生方法、新增原生方法等，其基本原理用一句话概括：JS传递字符串给OC，OC通过runtime接口调用或替换OC方法。下面主要针对方法调用、方法替换、新增方法来对JSPatch的原理做简单剖析。

#### 方法调用

##### 核心思想
JSPatch是怎么实现对JS的调用转换到OC的调用的？核心思想就是借助JavaScriptCore将类名字符串和方法名字符串传递给OC，由OC借助runtime来反射出类和方法来调用。以创建一个UIView实例的方法为例：

![2](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/2.png)

实现JS端 `UIView.alloc()`  到  OC端 `[UIView alloc]` 的映射调用。


##### JS构建对象、函数
然而JS的方法调用规则是必须对已经存在的对象调用已经存在的方法，因此如果直接调用``UIView.alloc()``，解析器会直接报错，因为当前JS上下文中UIView对象和alloc方法都不存在。既然如此，就去构建相应的对象和方法嘛。

构建对象没问题，只要在调用方法前构建相应的对象就行了。JSPatch的做法是在方法调用前，通过require函数为每一个类在JS中构建同名全局对象，JSPatch.js中相关源码如下：

```javascript
var _require = function(clsName) {
    if (!global[clsName]) {
      global[clsName] = {
        __clsName: clsName
      }
    } 
    return global[clsName]
}
global.require = function(clsNames) {
    var lastRequire
    clsNames.split(',').forEach(function(clsName) {
      lastRequire = _require(clsName.trim())
    })
    return lastRequire
}
```

对象构建好了，另一个问题就是方法的构建。作者一开始的思路是这样的：JS端把相关类名字符串传给OC端，OC端利用runtime取出类中所有方法，将所有方法名打包返回给JS端，JS端再为每个方法名构建同名函数。大致思路如下图：

![](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/3.png)

整个过程就是这样，听起来似乎没什么问题，唯一的问题就是：OC中，一个类可能就有成百上千个方法，JS为每个方法都构建相应的函数，内存暴涨！

由于内存消耗问题严重，上述思路pass。后来，作者痛定思痛，脑洞大开，从被JS规则约束的惯性思维里跳了出来：我们的最终目标是要让OC端调用指定的方法，JS端调用什么方法根本无所谓，只要在调用的方法中能够把类名和方法名传递给OC端就好了。所以，根本不需要为每一个方法定义JS函数，只要定义一个元函数，将JS端任意方法的调用，都替换成调用这个元函数，并将方法名作为参数传入这个元函数，在元函数中将类名、方法名以及参数传递给OC由OC来调用就万事大吉了！

这种方法，避免了为一个类的每个方法在JS端构建相应的函数，只需定义一个元函数即可，性能提升得不是一点半点！查看源码可以发现，我们的脚本JS代码在交由JavaScriptCore执行前，是先经过转换的，所有的方法调用都被替换成了调用__c函数：

![](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/4.png)

JS源码的这种转换实现很简单，是通过正则匹配替换掉的，核心源码如下：

```javascript
NSString *formatedScript = [NSString stringWithFormat:@";(function(){try{\n%@\n}catch(e){_OC_catch(e.message, e.stack)}})();", [_regex stringByReplacingMatchesInString:script options:0 range:NSMakeRange(0, script.length) withTemplate:_replaceStr]];
```

替换前的JS源码：

```javascript
require('UIColor,UIImage');
defineClass('CustomCell', {
    configWithModel: function(model) {
        self.headView().layer().setCornerRadius(5.0);
		self.headView().layer().setBorderColor(UIColor.darkGrayColor().CGColor());
        self.headView().layer().setBorderWidth(1.0);
        self.headView().layer().setMasksToBounds(YES);
        self.headView().setImage(UIImage.imageNamed(model.imgPath()));
        self.contentLabel().setText(model.content());
        self.contentLabel().setNumberOfLines(0);
    },
});
```

替换后的JS代码(所有的方法调用都被替换成了``__c``函数调用并将方法名作为参数传入)：

```javascript
;(function(){try{
require('UIColor,UIImage');
defineClass('CustomCell', {
    configWithModel: function(model) {
        self.__c("headView")().__c("layer")().__c("setCornerRadius")(5.0);
        self.__c("headView")().__c("layer")().__c("setBorderColor")(UIColor.__c("darkGrayColor")().__c("CGColor")());
        self.__c("headView")().__c("layer")().__c("setBorderWidth")(1.0);
        self.__c("headView")().__c("layer")().__c("setMasksToBounds")(YES);
        self.__c("headView")().__c("setImage")(UIImage.__c("imageNamed")(model.__c("imgPath")()));
        self.__c("contentLabel")().__c("setText")(model.__c("content")());
        self.__c("contentLabel")().__c("setNumberOfLines")(0);
    },
});
```
##### JS->OC消息传递

OK！现在JS端代码能执行了，按照之前说明，要在元函数也就是__c函数里将类名、方法名及参数传递给OC调用，具体是怎么实现的呢？其实前面已经提到过，是借助于JavaScriptCore实现的。

源码中``__c``函数（为便于理解，只抓取核心代码）

```javascript
__c: function(methodName) {
  var slf = this
  ... //Omit code
  return function(){
    var args = Array.prototype.slice.call(arguments)
    return _methodFunc(slf.__obj, slf.__clsName, methodName, args, slf.__isSuper)
  }
}
```

``_methodFunc``函数

```javascript
var _methodFunc = function(instance, clsName, methodName, args, isSuper, isPerformSelector) {
	...//Omit code
    var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
                         _OC_callC(clsName, selectorName, args)
    return _formatOCToJS(ret)
}
```

内部实现根据是实例方法还是类方法调用了``_OC_callI``和``_OC_callC``中的其中一个，然而发现，JS源码中并没有定义这两个函数，这是怎么回事？事实上，这两个函数在初始化JPEnige的时候就已经注册到JS上下文了。


```javascript
//JPEngine.m
+ (void)startEngine
{
	...
    context[@"_OC_callI"] = ^id(JSValue *obj, NSString *selectorName, JSValue *arguments, BOOL isSuper) {
        return callSelector(nil, selectorName, arguments, obj, isSuper);
    };
    context[@"_OC_callC"] = ^id(NSString *className, NSString *selectorName, JSValue *arguments) {
        return callSelector(className, selectorName, arguments, nil, NO);
    };
    ...
}
```

这是JavaScriptCore的接口，在JS上下文中创建JS函数。当函数被调用，会将消息传递给OC端，同时将参数传递给OC，OC执行相应的block，最后将返回值回传JS。

借助JavaScriptCore，JS的消息就能很好的传递给OC。

##### OC端方法调用
OC从JS端接收了消息，需要调用指定方法。JSPatch在处理的时候是通过NSInvocation来调用的，这是因为：JS传过来的参数类型需要转换成OC相应的类型，而NSInvocation很方便从方法签名中获取方法参数类型。同时，也能根据返回值类型取出返回值。

#### 方法替换

##### 基础原理
JS中通过一个``defineClass()``函数就能对OC中的方法进行替换，核心也是把类名和方法名传递给OC，由OC利用runtime进行方法的替换。首先要知道OC中类及方法在底层实现是均以结构体的形式存在的：

```objective-c
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class                                        OBJC2_UNAVAILABLE;
    const char *name                                         OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;

struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```

每个方法由3部分组成：SEL方法名、方法参数及返回值类型type、方法实现函数指针IMP。当一个方法被调用，会在该方法对应类的结构体的方法链表中遍历所有方法，匹配方法的SEL（事实上会先在缓存cache中查找方法）。一旦SEL成功匹配，就根据该SEL对应的函数指针IPM调用方法实现。若在该类的方法链表中没有匹配到方法，进而进入消息传递或者消息转发。

##### 原始方案（有坑）
OC中的方法允许动态替换，也就是说，可以将一个方法的SEL对应的IMP替换成一个新的IMP，也可以用一个新的SEL对应已知的某个IMP，runtime有相应的接口来实现方法替换。因此方法替换必然是借助runtime的这种性质实现，但是作者在具体实现的过程中也踩了不少坑，初始的思路是这样的（以Demo中替换CustomCell类中的configWithModel方法为例）：

![](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/5.png)

对于方法``configWithModel:``，定义一个新的方法实现（IMP）``configWithModelIMP：``,并用该IMP替换掉``configWithModel:``方法的IMP，使得``configWithModel:``方法的SEL与这个新的IMP对应。另外，创建一个新的方法名``ORIGconfigWithModel:``令其与``configWithModel:``的原始IMP对应起来。这样一来，当调用``configWithModel:``方法时，实际调用的替换过的方法实现``configWithModelIMP：``，只要在该方法实现中根据JS传过来的参数调用JS实现就好了；同时JS端也能够通过``ORIGconfigWithModel：``来调用``configWithModel:``的原始实现了。

这种思路还有一个问题没解决，就是新建的IMP参数要怎么获取？对于一个方法，参数是确定的，写对应的IMP没有问题：

```objective-c
static void configWithModelIMP (id slf, SEL sel, CustomModel *model) {
   [function callWithArguments:@(model)];  //执行JS实现
}
```

但是我们不可能为每一个原始方法都新构建一个对应的IMP，而是只会构建一个IMP，让所有方法调用都走这个IMP。因此就要有一种通用的方法，能够获取到不同方法的参数。作者最先想到的办法就是通过可变参数``va_list``来实现，大致实现如下：

```objective-c
static void commonIMP(id slf, ...)
  va_list args;
  va_start(args, slf);
  NSMutableArray *list = [[NSMutableArray alloc] init];
  NSMethodSignature *methodSignature = [cls instanceMethodSignatureForSelector:selector];
  NSUInteger numberOfArguments = methodSignature.numberOfArguments;
  id obj;
  for (NSUInteger i = 2; i < numberOfArguments; i++) {
      const char *argumentType = [methodSignature getArgumentTypeAtIndex:i];
      switch(argumentType[0]) {
          case 'i':
              obj = @(va_arg(args, int));
              break;
          case 'B':
              obj = @(va_arg(args, BOOL));
              break;
          case 'f':
          case 'd':
              obj = @(va_arg(args, double));
              break;
          …… //其他数值类型
          default: {
              obj = va_arg(args, id);
              break;
          }
      }
      [list addObject:obj];
  }
  va_end(args);
  [function callWithArguments:list];
}
```

这样一来，不管有多少参数都能通过``va_list``取出来，最终组成一个数组传给JS调用。似乎参数的问题的已经解决了，但是按照苹果的尿性，你永远也不知道自己有多少坑要踩。事实上，这段代码在arm64架构的机子上跑，程序就会崩溃，原因是arm64上``va_list``的结构改变了，导致无法像上面这样取参数。不过正是这样一个坑，迫使作者催生出了一种黑科技般的方法替换方案。

##### 新方案（完美）
新的方法替换方案是通过runtime的消息转发机制实现的。关于消息转发，前面稍微提到过，简而言之就是当被调用的方法在其类及其父类上都不存在时，会将消息进行转发。消息转发有3个步骤,可以说有3种方式：

+ resloveInstanceMethod
+ forwardingTargetForSelector
+ forwardingInvocation

这里是借助于消息转发的第三步即``forwardingInvocation``来实现的。另外要知道的是``_obj_msgforward``这个东西，它是一个IMP，当调用的方法在消息传递过程中没有匹配时，就会调用``_obj_msgforward``这个实现，由它来执行消息转发。如果我们手动调用``_obj_msgforward``这个实现，就会跳过消息传递过程，直接进入消息转发，这样就能避开方法初始实现的调用，在消息转发中调用自定义的方法实现，这就是这种方案最核心的思想。同时关于参数问题，``forwardingInvocation``方法会传进一个NSInvocation参数，可以从中获取到方法调用的所有信息，包括方法名、参数及返回值类型、参数值等，因此不存在先前那种方案的无法获取参数的问题。

整体的实现方案如下：

![](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/6.png)

通过这种方案，当调用``configWithModel:``方法时，会调用``_obj_msgForward``这个IMP，该方法内部实现会调用``forwardInvocation:``方法，而``forwardInvocation:``方法的实现已被替换成``JPForwardInvocation:``实现，因此最终会调用``JSForwardInvocation:``。在这个函数内部，要做的就是根据传进来的NSInvocation实例获取到所有参数，然后传给JS调用。

另外可以看到，对于被替换的方法的实现，都会创建一个Origin方法名与其对应，以便通过这个新的方法名能够调用到原始的方法实现。

相关源码见*JPEngine.m*中  `overrideMethod` 方法的具体实现。

最后一个问题，把``forwardInvocation:``方法hook了，程序中若有其他的消息转发需求，不会被影响吗？是的，所有的消息转发都会走到``JPForwardInvocation``，因此在这个函数实现里首先要做的事就是判断当前转发的消息是否是我们调用的消息，如果是，就走我们的逻辑；如果不是，就通过``ORIGforwardInvocation``方法调用原始的``forwardInvocation``方法实现。

另外需要注意的是，如果替换的是协议中的方法，需要在``defineClass``接口里的类描述参数里将协议名及父类名写进去，写法同OC申明类接口一致，这样当在类中找不到方法时会去到协议中查找。

#### 新增方法
在方法替换中，我们将要替换的方法在``defineClass``中定义，然后将方法的参数个数、方法实体打包成一个数组传递给OC端，由OC来执行方法替换。事实上，新增方法在JS端并没有多余的处理，同样是在``defineClass``中定义要新增的方法，JS端只负责把相关数据传给OC端，具体的逻辑完全在OC端处理。

OC端处理的逻辑大致是这样的：判断方法是否存在于类中？若存在，替换；若不存在，判断方法是否存在于当前类遵守的协议当中，若存在则从协议的方法申明中获取方法参数类型编码，然后新增方法；若不存在，将方法所有参数及返回值类型设为id类型，然后新增方法。巧就巧在作者对于新增方法的处理上：并不是为每个方法创建对应的实现，而是将所有要新增的方法的SEL与``_obj_msgForward``这个IMP对应起来然后添加，这样调用新增的方法，逻辑跟方法替换中的逻辑是一样的，实际上进行的是消息转发，执行``JPForwardInvocation:``方法。只要将JS传过来的方法体通过一个全局字典保存，在``JPForwardInvocation:``中根据方法SEL取出对应的JS方法体执行即可。这种做法，避免了创建新的方法实现增加消耗，又能将方法替换的逻辑和新增方法的逻辑完美地结合起来，猴赛雷！

![](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/7.png)

### JSPatch安全问题

JSPatch通过下发JS脚本文件对app进行修复或更新，经过刚才的分析，JS脚本的权限是很大的，如果在下发传输过程中文件被第三方截获，修改了脚本内容，那么对app以及用户数据可能会造成致命的伤害。因此，必须制定一个安全可靠的方案保证JSPatch热修复的脚本文件传输的安全。

毫无疑问，要对脚本文件进行加密，大致有3套方案：

1. 对称加密。服务器端和客户端保存一把相同的私钥，下发脚本文件前先对文件进行加密，客户端拿到脚本文件后用相同的私钥解密。这种方案弊端很明显，密钥保存在客户端，一旦客户端被破解，密钥就泄露了。
2. https传输。这种方案安全可靠，但是成本较大，需要部署服务器，购买证书，对于一个小型的app来说门槛较高，当然如果不介意成本那么这种方案是非常安全的。
3. RSA签名验证。借助于RSA非对称加密，这种方案在保证安全性的同时，门槛低，成本小，通用性强，对客户端和服务端都没什么特殊要求。

RSA签名验证整体流程如下：

![](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/8.png)

服务器端：

1. 对要下发的脚本文件计算MD5值
2. 用服务器私钥对MD5值进行加密
3. 将脚本文件和加密后的MD5值下发给客户端

客户端：

1. 用服务器端公钥解密加密过的MD5值
2. 对接收的脚本文件计算MD5值
3. 将解密出来的MD5与新计算出来的MD5进行比对校验，若校验通过，则表明脚本在传输过程中没有被篡改。

作者针对RSA这套安全方案，制作了相关组件[JPLoader](https://github.com/bang590/JSPatch/tree/master/Loader)，客户端可以直接集成。同时也开放了一个管理下发脚本文件的平台[JSPatch平台](http://jspatch.com/)，可以直接使用这个平台进行脚本下发及版本的管理，但是该平台提供的服务是按照请求量收费的。

### 开发效率

+ JSPatch Convertor （[http://bang590.github.io/JSPatchConvertor/](http://bang590.github.io/JSPatchConvertor/)）
	OC->JS代码转换工具，支持大部分OC语法的转换，但部分细节转换还不支持，例如宏定义、枚举值、静态变量等，需要手动进行转换。但是对于有JS短板的开发者来说，这个工具还是非常实用的。
	
+ JSPatchX （[https://github.com/bang590/JSPatchX](https://github.com/bang590/JSPatchX)）

	代码自动补全插件，在手动编写JS代码时是非常实用的工具。
	
+ JSPatchPlaygroundTool ([https://github.com/Awhisper/JSPatchPlaygroundTool](https://github.com/Awhisper/JSPatchPlaygroundTool))
	
	编码时无需重启模拟器，每次修改脚本后刷新可以实时看到修改后的变化。主要原理是将原先替换过的函数还原，然后重新执行JS，即可达到reload的效果。

## 总结

### 主流方案
之前提到的几种热修复的方案中，WebView嵌套h5的方案完全属于web端范畴、动态下发动态库的方案由于苹果的态度含糊不清，以及通过动态下发lua脚本的WaxPatch方案由于年久失修几乎已经被JSPatch替代，暂时排除这3种方案，那么当下iOS热修复的方案主要剩下这三种：

+ JSPatch
+ React Native
+ Weex

又由于React Native和Weex在底层实现原理上是一脉相承的，并且Weex刚开源没多久，暂时还没有得到广泛推广和接纳，因此将RN和Weex并到一起与JSPatch进行对比。

### JSPatch & RN/Weex 方案对比
#### 底层原理
+ JSPatch: 主要通过runtime来调用和替换native方法，借助于JavaScriptCore把JS的调用映射到native调用上。
+ RN/Weex: ReactNative和Weex的底层原理是一脉相承的，但和JSPatch是完全不同的。以React Native为例，native需要开发相关的组件或API给JS，JS才能调用。关于JS与native的交互，RN有它自己的一套通信机制。

#### 学习成本
项目中接入一门技术方案，团队对于接纳这套技术方案的学习成本也应当纳入评估范围。

+ JSPatch: 对于native开发者来说，它的学习成本是非常小的，只要稍微学一下JS即可。事实上借助于JSPatch Convertor，即便只是略懂JS皮毛，也不会成为太大的障碍。
+ RN/Weex: React Native的语法结构是基于前端框架React.js的，Weex基于前端框架Vue.js，因此对于前端开发人员是非常友好的，而对于像我这种前端小白来说，还是先去学一轮前端三剑客再说吧！另外即便是前端开发者，要想吃透React Native，终端开发技能也是必不可少的。因为前面说过，RN只能基于native开放给JS的组件或者API进行运作，一旦出现较为深度的bug，或者现有的组件无法满足需求，就需要开发人员深入到native端去解决问题。而React Native又是一个跨平台的框架，跨安卓和iOS，这样一来吃透React Native，就是吃透三端了！

#### 接入成本

+ JSPatch: JSPatch初始定位就是为hotfix而生的，完全是从终端出发设计的一套方案，整个框架一共就3个文件不超过3K行代码，方便接入不说，接入后对整个包的体积的影响也是微乎其微的，只占100K左右。
+ RN/Weex: RN的设计出发点并不是为了hotfix，它的野心颇为宏大，剑指整个APP,即用前端开发方式去开发原生APP！而Weex的野心就只能用颤抖来形容了，那就是：一统三端！正所谓“心有多大，舞台就有多大”，RN/Weex的舞台确实是大了点，要搭建一整套的环境支持，添加很多的依赖库。Weex不清楚，RN在接入后整个包的体积增加了2M左右，当然还取决与你要用到的组件。

#### 开发效率

+ JSPatch: 仅针对iOS平台，采用原生开发方式，前面提到过一些提升开发效率的一些工具。
+ RN/Week: 跨平台。就RN而言，逻辑层代码跨平台复用，UI层根据不同平台写不同代码，但是通过一些工具也能做到跨平台。采用的是前端开发方式，相比于JSPatch的终端开发方式，萝卜青菜各有所好。

#### 热修复能力

+ JSPatch: 因为其底层借助于runtime动态调用和替换方法来实现，因此可以对任意native代码进行hotfix。
+ RN/Weex: 只能借助已有的组件或API，对使用RN/Weex编写的模块进行hotfix。因此相比于JSPatch，热修复能力有一定的局限性。

#### 性能对比
JSPatch和RN/Weex性能都较高，相比于纯原生而言，RN/Weex可能会稍逊一筹，但是相比Hybrid，肯定要高出一截。总体而言，两者性能相差不大，各有各的消耗点。

+ JSPatch：主要消耗在JS与OC的通信上，每次JS调用，要经过参数包装、JS引擎传递消息、类名方法名反射、参数类型转换、方法调用、返回值类型转换、回传返回值。
+ RN： RN对于JS与OC间的通信优化做得比较好，不是主要性能消耗点。它的性能主要消耗在框架本身的模块初始化、组件初始化以及JS渲染逻辑等上面。

### 对比结论

![](http://oubmw34rc.bkt.clouddn.com/blog/jspatch/9.png)

就iOS原生应用的热修复方案而言，JSPatch是首选方案；就采用了React Native或Weex开发的模块，其自带的热修复能力已经非常强大。但是如果纯碎为了RN/Weex的热修复能力，而将模块开发方案采用从原生转向RN或Weex是毫无必要的，如果真的要转，那我相信更多的是基于RN/Weex框架本身的跨平台能力。
