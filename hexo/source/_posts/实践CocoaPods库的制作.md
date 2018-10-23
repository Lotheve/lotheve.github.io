---
title: 实践CocoaPods库的制作
date: 2018-10-23 14:42:38
tags: iOS
---

> 本篇内容基于 CocoaPods V1.6.0 实践公有库及私有库的制作

## 前言

作为一名iOSer，我们知道CocoaPods的作用是帮助我们管理和维护代码仓库。在说具体的如何制作Pod仓库之前，需要先来了解一下Pods是如何从远端拉取代码的。

```
~/.cocoapods/repos
```

 安装了Cocoapods之后，本地会有这个路径，默认里面只有一个 `master` 目录

	.
	├── master

这个 `master` 目录其实是一个仓库，在该目录下执行 `git remote -v` ，就能看到该目录关联的远端仓库：

```bash
$ git remote -v
origin	https://github.com/CocoaPods/Specs.git (fetch)
origin	https://github.com/CocoaPods/Specs.git (push)
```

![](http://oubmw34rc.bkt.clouddn.com/blog/pods/1.png)

这是CocoaPods官方用来存放公有库版本描述文件的仓库，何为代码描述文件？继续往下走，能看到 `master` 仓库下深藏玄机。

![](http://oubmw34rc.bkt.clouddn.com/blog/pods/2.png)

可以看到目录下包含了数不清的各个版本的公有库的目录，每个目录下都有一个 `.podspec.json` 文件。这个文件就是所谓的描述文件，里面包含了某个版本的公有库代码的具体描述信息，包括代码源仓库的路径、版本号、作者等。当我们执行 `pod install` 需要更新代码时，就是根据该文件下的 `source` 指定的路径在拉取源代码的。

我们在发布代码库时，不管是公有库还是私有库，都需要两个仓库：1.存放源代码的仓库 2.存放版本描述文件的仓库。对于公有库而言，后者就是官方的Specs仓库，因此不需要手动创建。并且每次发布公有库时，CocoaPods已经给我们提供了相关的命令来提交描述文件到Specs仓库上了。而对于私有库而言，这两个仓库则都需要我们手动创建和维护。

## 公有库

### 注册CocoaPods账号

要想发布公有库，需要注册CocoaPods，Pods提供了相应的注册命令：

```bash
$ pod trunk register EMAIL [YOUR_NAME]
```

+ `EMAIL`：注册的邮箱
+ `YOUR_NAME`：注册的姓名

执行后，CocoaPods会发送一份验证邮件到你指定的邮箱上，登陆邮箱进行验证。

```
[!] Please verify the session by clicking the link in the verification email that has been sent to youremail
```

 ![](http://oubmw34rc.bkt.clouddn.com/blog/pods/3.png)

点开邮箱中的链接，页面显示如上即为注册成功，此时可在终端查看注册信息。

```bash
$ pod trunk me
```

显示如下：

	- Name:     Lotheve
	- Email:    gaofeng-7171@163.com
	- Since:    October 19th, 01:04
	- Pods:     None
	- Sessions:
	  - October 19th, 01:04 - February 24th, 2019 01:14. IP: 101.71.41.233	

### 创建公有仓库

仓库中包含以下文件：	

```
├── LICENSE
├── LTVMediaPicker
├── LTVMediaPicker-Demo
├── LTVMediaPicker.podspec
└── README.md
```

+ `LICENSE`：开源许可证
+ `README.md`：库描述
+ `LTVMediaPicker`：库源码
+ `LTVMediaPicker-Demo`：demo工程，非必须
+ `LTVMediaPicker.podspec`：库描述文件，**重要！**

### .podspec描述文件创建

在仓库目录下执行下面命令即可生成配置文件：

```bash
$ pod spec create LTVMediaPicker
```

生成的文件中包含了很多注释及默认的配置项，根据需要整理后如下保留如下配置即可：

```ruby
Pod::Spec.new do |spec|
  # 项目名称
  spec.name         = "LTVMediaPicker"            
  # 版本号
  spec.version      = "1.0.0"
  # 项目简介
  spec.summary      = "An media-asset picker from photo album or camera"
  # 项目描述
  spec.description  = <<-DESC
                      LTVMediaPicker is a simple tool to pick photo or video from album or camera.
                      DESC
  # 项目主页
  spec.homepage     = "https://github.com/Lotheve/LTVMediaPicker"
  # 开源协议
  spec.license      = "MIT"
  # 作者
  spec.authors            = { "Lotheve" => "gaofeng-7171@163.com" }
  # 作者主页
  spec.social_media_url   = "https://lotheve.github.io/"
  # 支持的平台及版本信息
  spec.platform     = :ios, "7.0"
  # 代码源地址 需使用https协议地址
  spec.source       = { :git => "https://github.com/Lotheve/LTVMediaPicker.git", :tag => "#{spec.version}" }
  # 源文件
  spec.source_files  = "LTVMediaPicker/*.{h,m}"
  # 需要链接的系统库
  spec.frameworks = 'Foundation', 'UIKit'
  # 是否需要ARC支持
  spec.requires_arc = true
end
```

方便起见也可以在官方提供的[配置示例](https://guides.cocoapods.org/syntax/podspec.html)上进行修改。

### 校验.podspec文件格式

```bash
$ pod lib lint
```

验证通过会提示

```
-> LTVMediaPicker (1.0.0)
LTVMediaPicker passed validation.
```

若验证失败也会给出相应的提示，根据提示修改即可。

### 仓库提交

.podspec验证成功后，即可将仓库提交到远程了，同时需要给提交节点打上标签。标签用于稳定存储版本，要与当前配置文件中的 `s.version` 版本号一致，设置为 `1.0.0` 。

```bash
$ git push
$ git tag -a 1.0.0 -m '1.0.0版本'
$ git push origin --tags
```

### 仓库发布

现在，只需将 `.podspec` 发布到公共Specs仓库中，即可完成仓库的发布。在仓库目录下执行：

```bash
$ pod trunk push LTVMediaPicker.podspec
```

若提示 `You need to register a session first.` ，是因为未注册CocoaPods账号，需要先注册。

这个过程做了如下操作：

+ 更新CocoaPods的本地Specs仓库（这个过程可能等待时间较久）
+ 将 `.podspec` 文件转换为 JSON 格式
+ 将本地版本库的修改推送远程Specs仓库

发布成功后，终端会给出如下信息：

	--------------------------------------------------------------------
	 🎉  Congrats
	 🚀  LTVMediaPicker (1.0.0) successfully published
	 📅  October 19th, 01:26
	 🌎  https://cocoapods.org/pods/LTVMediaPicker
	👍  Tell your friends!
	--------------------------------------------------------------------

现在，打开 [https://cocoapods.org/pods/LTVMediaPicker](https://cocoapods.org/pods/LTVMediaPicker) 就能看到发布的库了！

### 仓库使用

pod库发布后，若执行 `pod search LTVMediaPicker` 搜索不到结果，提示

```
[!] Unable to find a pod with name, author, summary, or description matching `LTVMediaPicker`
```

则需要清楚一下本地搜索索引缓存：

```bash
$ pod setup
$ rm ~/Library/Caches/CocoaPods/search_index.json
```

然后再执行

```bash
$ pod search LTVMediaPicker
```

搜索结果如下：

	-> LTVMediaPicker (1.0.0)
	   An media-asset picker from photo album or camera
	   pod 'LTVMediaPicker', '~> 1.0.0'
	   - Homepage: https://github.com/Lotheve/LTVMediaPicker
	   - Source:   https://github.com/Lotheve/LTVMediaPicker.git
	   - Versions: 1.0.0 [master repo]

然后就可以install了。

### 仓库维护

当公有库代码更新后，需要发布迭代版本，步骤很简单，只需：

1. 更新 `LTVMediaPicker.podspec` 中的仓库版本号
2. 更新代码并推送更新到远程，同时打上版本号标签
3. 执行 `pod trunk push LTVMediaPicker.podspec` 

现在我对LTVMediaPicker做一点小小的修改，代码不更新，仅是将库支持的系统最低版本修改为 `iOS8.0` 

```ruby
spec.platform     = :ios, "8.0"
```

修改后按上述步骤操作，发布 `1.0.1` 版本。

发布后再执行 `pod search LTVMediaPicker` ，搜索结果如下：

	-> LTVMediaPicker (1.0.1)
	   An media-asset picker from photo album or camera
	   pod 'LTVMediaPicker', '~> 1.0.1'
	   - Homepage: https://github.com/Lotheve/LTVMediaPicker
	   - Source:   https://github.com/Lotheve/LTVMediaPicker.git
	   - Versions: 1.0.1, 1.0.0 [master repo]

可以看到 `1.0.1` 版本已经成功发布了，或者执行 `pod trunk info LTVMediaPicker` ，也能看到对应的库信息：

	LTVMediaPicker
	- Versions:
	  - 1.0.0 (2018-10-19 07:26:52 UTC)
	  - 1.0.1 (2018-10-19 11:16:15 UTC)
	- Owners:
	  - Lotheve <gaofeng-7171@163.com>

### 仓库版本删除

```
pod trunk deprecate [POD REP NAME]	
```

该命令可以使整个库过期。

```bash
pod trunk delete [POD REP NAME] [VERSION]
```

该命令用来删除已发布的指定版本的仓库，并且不可回退。一旦删除，该版本将无法再创建。

## 私有库

### 创建私有描述文件仓库

文初提过，私有库的制作需要开辟两个仓库，一个是用来存放代码的，另一个是用来存放版本描述文件的（对于公有库而言，这个仓库就是官方的Specs仓库，对应本地 `~/.CocoaPods/repos/` 目录下的 `master` 仓库）。既然是私有库，就要选择能够创建私有库的代码托管平台，我这里选择Coding作为实践（提供免费私有仓库）。github自然也是可以创建私有仓库的，只不过要收费。创建存放代码描述文件的私有仓库：

![](http://oubmw34rc.bkt.clouddn.com/blog/pods/4.png)

创建成功之后，执行

```bash
$ pod repo add PrivateRepo git@git.coding.net:lotheve/LTVRepo.git
```

将远程的私有描述文件仓库关添加到本地。这时候再来看 `~/.cocoapods/repos` 目录下的文件：

	.
	├── PrivateRepo
	└── master
发现本地已经增加了 `PrivateRepo` 目录，对应的就是远端的 `LTVRepo` 仓库。

后续若想移除该私有库，执行下面指令即可：

```bash
$ pod repo remove PrivateRepo
```

### 创建私有代码库

![](http://oubmw34rc.bkt.clouddn.com/blog/pods/5.png)

### 添加代码文件、.podspec文件

1. 新建一个文件夹LTVGeneral，添加要打包到私有库的代码。

2. 创建 `.podspec` 文件

   ```bash
   $ pod spec create LTVGeneral
   ```

此时文件目录结构如下，其中LTVGeneral下的文件为私有库代码文件

	.
	├── LICENSE
	├── LTVGeneral
	│   ├── LTVDateTool.h
	│   └── LTVDateTool.m
	├── LTVGeneral.podspec
	└── README.md

修改 `.podspec`  文件内容，按照前面公有库介绍的模板配置即可：

```ruby
Pod::Spec.new do |spec|
  # 项目名称
  spec.name         = "LTVGeneral"            
  # 版本号
  spec.version      = "1.0.0"
  # 项目简介
  spec.summary      = "A private code library"
  # 项目主页
  spec.homepage     = "https://coding.net/u/lotheve/p/LTVGeneral"
  # 开源协议
  spec.license      = "MIT"
  # 作者
  spec.authors            = { "Lotheve" => "gaofeng-7171@163.com" }
  # 支持的平台及版本信息
  spec.platform     = :ios, "8.0"
  # 代码源地址
  spec.source       = { :git => "git@git.coding.net:lotheve/LTVGeneral.git", :tag => "#{spec.version}" }
  #spec.source       = { :git => "https://git.coding.net/lotheve/LTVGeneral.git", :tag => "#{spec.version}" }
  # 源文件
  spec.source_files  = "LTVGeneral/*.{h,m}"
  # 需要链接的系统库
  spec.frameworks = 'Foundation', 'UIKit'
  # 是否需要ARC支持
  spec.requires_arc = true
end
```

然后验证仓库配置的正确性

```bash
$ pod lib lint
```

这时报了一个警告：

```
- WARN  | source: Git SSH URLs will NOT work for people behind firewalls configured to only allow HTTP, therefore HTTPS is preferred.
[!] LTVGeneral did not pass validation, due to 1 warning (but you can use `--allow-warnings` to ignore it).
```

说是source不建议用git协议。其实这里用git协议或者https协议的地址都可以，只是使用前者时，团队其他成员安装私有库依赖还需要先在代码仓库上部署公钥，这有助于进一步控制团队内的“私有”（如果你确定有这必要的话）。若确定使用git协议，指令后面加上 `--allow-warnings` 参数即可忽略警告。此处我选择https，修改配置：

```ruby
  #spec.source       = { :git => "git@git.coding.net:lotheve/LTVGeneral.git", :tag => "#{spec.version}" }
  spec.source       = { :git => "https://git.coding.net/lotheve/LTVGeneral.git", :tag => "#{spec.version}" }
```

再次验证，输出验证通过（LTVGeneral passed validation）。

### 推送代码和描述文件

现在，可以把代码推送到远程了，记得打上版本标签

```bash
$ git push
$ git tag -a 1.0.0 -m '1.0.0版本'
$ git push origin --tags
```

接下来，推送版本描述文件到远程仓库，命令如下：

```bash
$ pod repo push PrivateRepo LTVGeneral.podspec
```

这个过程经历了如下步骤：

+ Validating spec

  验证 `.podspec` 文件

+ Updating the 'PrivateRepo' repo 

  更新本地版本描述文件仓库

+ Adding the spec to the 'PrivateRepo' repo

  添加新的描述文件到本地描述文件仓库中

+ Pushing the 'PrivateRepo' repo

  推送本地描述文件仓库的修改到远程

这时候进到本地Pods目录下，查看PrivateRepo的内容如下：

	├── LICENSE
	├── LTVGeneral
	│   └── 1.0.0
	│       └── LTVGeneral.podspec
	└── README.md
可以看到1.0.0版本的LTVGeneral描述文件已经存在了。

### 工程安装私有库

首先search一下私有库看下：

```bash
$ pod search LTVGeneral
-> LTVGeneral (1.0.0)
   A private code library
   pod 'LTVGeneral', '~> 1.0.0'
   - Homepage: https://coding.net/u/lotheve/p/LTVGeneral
   - Source:   https://git.coding.net/lotheve/LTVGeneral.git
   - Versions: 1.0.0 [PrivateRepo repo]
```

没问题。

现在，在先前的测试工程的 `Podfile` 中添加 `pod 'LTVGeneral', '~> 1.0.0'` 来准备安装私有库。然后执行 `pod install` , 可以看到如下报错：

```
[!] Unable to find a specification for `LTVGeneral (~> 1.0.0)`
```

这是因为CocoaPods默认从官方的Specs仓库中查找目标库的信息，若要从私有库中查找，需要在 `Podfile` 中指定源地址。如果是安装私有库的同时，还要安装公有库，那么公有库的版本仓库地址也要显式加上。修改后的 `Podfile` 文件如下：

```ruby
source 'https://git.coding.net/lotheve/LTVRepo.git'
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, '8.0'

target 'testProject' do
pod 'LTVMediaPicker', '~> 1.0.0'
pod 'LTVGeneral', '~> 1.0.0'
end
```

需要注意的是source指定的是版本描述文件的仓库地址，而非代码库的仓库地址。随后再次执行 `pod install` ，即可安装成功。`pod install` 经历了如下过程：

1. 根据 `podfile` 中指定的source路径拉取版本库到本地
2. 从拉取到的版本库中查找目标库的 `.podspec` 文件
3. 根据查找到的 `.podspec` 中指定的source路径拉取代码，随后整合到工程中

现在打开工程，可以看到依赖已经安装了：

![](http://oubmw34rc.bkt.clouddn.com/blog/pods/6.png)

### 私有库的更新

关于私有库的更新，其实和公有库是一样的流程：

1. 更新代码，修改 `.podspec` 中的版本号，打上版本标签，推送到远程代码仓库
2. 执行 `pod repo push PrivateRepo LTVGeneral.podspec`  发布新版本

## 结语

通过对公有库制作和私有库制作的实践，相信大家（包括我本人）对CocoaPods的理解更透彻了。实际上对于一些体量较小的工程而言，大多会使用CocoaPods接入一些三方公有库，当然也有不少公司维护内部的私有组件库来实现快速开发。而对于一些大工程而言，则能够充分利用CocoaPods的私有库能力来实现工程的组件化，实现模块之间的解耦。接下来我将对工程组件化进行探索~