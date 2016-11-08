# CocoaPods 私有公有库的搭建

## 前期准备

### 第一步需要注册一个 cocoapods 账号

命令为
```
pod trunk register 邮箱 '昵称' --description='设备信息'
```
例如:

```
pod trunk register heronlyj@gmail.com 'EugeneLi' --description='My MacBookPro13' 
```

邮箱会受到邮件，然后验证。然后回来检查自己是否注册成功

```
pod trunk me
```

账号跟设备是绑定的，并没有登录这一说法，因此如果你切换设备之后，可以再次注册一次，只需 description 可以描述一下你当前的设备信息，昵称就可以不用写了


### 代码

这里有两种情况，一种是已经写好了代码，一种是还没有

#### 已经写好代码的

直接在根目录下执行如下命令， `QuickBuild` 就是让别人看到的开源项目名称

```
pod spec create 'QuickBuild' 
```

#### 如果还没有创建好项目的

直接执行这个命令

```
pod lib create 'QuickBuild'
```

运行完这个目录后，`Cocoapods` 会自动帮你产生一个包含一部分基础内容的 `Podspec` 文件，通过对里面进行一些必要的修改即可

### Podsepc 文件 （*.podspec）

这个文件是告诉 Cocoapods 你这个库的一些基本信息，包括你的版本号、获取的地址、哪些文件是希望被包含进来的等等

```
Pod::Spec.new do |s|  
	s.name         = "QuickBuild"
	s.version      = "1.0"
	s.summary      = "A custom framework with base component."
	s.homepage     = "https://github.com/heronlyj/QuickBuild"
	s.license      = { :type => "MIT", :file => 'LICENSE.md' }
	s.author       = { "EugeneLi" => "heronlyj@gmail.com" }
	s.platform     = :ios, "8.0"
	s.source       = { :git => "https://github.com/heronlyj/QuickBuild.git", :tag => s.version}
	s.frameworks   = 'UIKit'
	s.source_files = "QuickBuild/**/*.{h,m,swift}"
	s.resource     = "QuickBuild/Resources/**"
	s.requires_arc = true
end

s.subspec 'Security' do |ss|  
	ss.source_files = 'AFNetworking/AFSecurityPolicy.{h,m}'
	ss.public_header_files = 'AFNetworking/AFSecurityPolicy.h'
	ss.frameworks = 'Security'
end 
```
在 Podspec 文件中，很经常看到 do |名字| 的形式，这个有点类似于在对应 end 之前声明了一个 local 的参数一样。除了上问所示的 Pod::Spec.new do |s| 的情况外，还有可能会出现在 s.subspec 'SubModule' do |sm| 的形式，同样 sm 也是可以自己修改成自己想要的名字.

`s.subspec 'Security' do |ss|` 就是声明的子模块, 对应的意思是 `父节点名.subspec '子模块名' do |子节点名|`

### 验证错误

```
pod lib lint QuickBuild.podspec --verbose --allow-warnings
```
这条命令是用来验证你的Podspec文件是否有问题，并且代码里面是否有问题的命令，后面可以才参数, 下面命令可以查看有哪些参数

```
pod lib lint --help
```

`--verbose` 这条参数可以输出所有的消息信息
`--allow-warnings` 是否允许警告，在用到第三方框架的时候，有的时候是自带会有warmings的代码，用这参数可以屏蔽警告

### 给项目打 tag

验证通过之后，就可以用 git 命令给自己的项目打 tag 了

```
git tag -a v1.0.0 -m "version 1.0.0 release"
```

### 上传

最后一步上传到 cocoapods 仓库

```
pod trunk push QuickBuild.podspec
```
之后可以验证一下

```
pod search QuickBuild
```

如果是多人维护, 如下命令可以添加小组人员

```
pod trunk add-owner ARAnalytics kyle@cocoapods.org
pod trunk add-owner '项目名' '邮箱'
```

## 私有仓库

上面的步骤是将开源库共享到 cocoapods 的公共仓库，但是有时候我们需要使用 cocoapods 给自己的私有库做模块化管理

### 创建私有仓库

私有仓库一般是 github 私有仓库，cording 私有仓库, 或者内网的 Gitlab 仓库。

格式
```
pod repo add '仓库名' '仓库地址'  
```

例如:
```
pod repo add 'eit' 'git@192.168.1.254:iOS/podsRepo.git'
```
新建的仓库在 `~/.cocoapods/repos` 可以看到多了一个 `eit` 文件夹

接下来的步骤跟开源仓库类似，配置好私有仓库对应的 Podsepc 文件，编写代码，验证正确性

最后上传这一步骤是不一样的将

```
pod trunk push 项目名.podspec
```

更换为

```
pod repo push '私有仓库名' 项目名.podspec
//
pod repo push 'eit' Quickbuild.podspec
```

### 使用

使用的时候，在项目根目录下的 PodFile 文件内，首先要声明仓库地址

```
# 公共仓库地址
source 'https://github.com/CocoaPods/Specs.git'

# 私有仓库地址 
source 'http://192.168.1.254:8800/iOS/podsRepo.git'
```

以后的就都一样了

### 开发模式

如果独立的模块处于开发阶段，代码频繁更新，这个时候每次 pod update 显然是不现实的，我们可以使用开发模式，直接引用本地代码，方式是在 `Podfile` 中按路径名来申明

```
pod '库名', :path => '本地路径'
```

但是对于如果修改了目录结构（添加、删除或者移动文件文件）或者是修改了 Podspec 文件的配置的话，最好是运行一下 pod update 的命令。普通修改代码的情况下就不需要运行 pod update 命令和打 tag 了。

```
pod 'QuickBuild', :path => '~/workspace/QuickBuild'  
```



### 最后需要注意的地方

普通使用私有库，直接在 `Podfile` 中增加一个 source 声明即可

```
# 公共仓库地址
source 'https://github.com/CocoaPods/Specs.git'

# 私有仓库地址 
source 'http://192.168.1.254:8800/iOS/podsRepo.git'
```

在私有库中引用私有库，即在Podspec文件中依赖 (dependency) 私有库 这种情况就比较麻烦一点，因为毕竟Podspec文件中并没有指明私有仓库地址的地方。那么肯定就不在 Podspec 文件里面指明私有仓库的地方。而是在验证和上传私有库的时候进行指明。即在下面这两条命令中进行指明：

```
pod lib lint 项目名.podspec --sources=https://github.com/CocoaPods/Specs.git,192.168.0.100:Plutoy/Specs.git
```
以及

```
pod repo push --source=https://github.com/CocoaPods/Specs.git,192.168.0.100:Plutoy/Specs.git,
```
要不然你在检验项目以及提交项目过程中就会出现Error的情况。

但是这两种情况还是有点不同的，第一种情况是可以采用开发者模式，而第二种情况不能采用开发者模式，只能通过打tag之后才能进行使用，所以在使用第二种情况下最好是测试好之后打完tag再进行引用。

还有以下几点
1. 模块中不能出现循环依赖，即 A 依赖 B，B 依赖 C，C 依赖 A 的情况
2. 如果一个子模块引用，需要填写完整的模块名，如在 Core 模块下面有 Controller 模块下面有个 Setting 模块，并且整个库的名字为 TestProj 的话,则依赖的名称需要这样写 s.dependency 'TestProj/Core/Controller/Setting' 的形式
3. 如果出现模块特别多的情况下，在验证过程中，竟然采用--subspec=子模块名来进行一个模块一个模块验证，特别是对于如果只改动了一个模块的情况下，这里所说的字模块名也和上面一点异样，要填写完整的模块名
4. 在写私有库的过程中，竟然不用 prefix header 的形式，因为在分子模块的过程中很容易出现忘记引用 header 而出现的 Error

----
参考

[Cocoapods系列教程(二)——开源主义接班人](http://www.pluto-y.com/cocoapods-contribute-for-open-source/)
[Cocoapods系列教程(三)——私有库管理和模块化管理](http://www.pluto-y.com/cocoapod-private-pods-and-module-manager/)

