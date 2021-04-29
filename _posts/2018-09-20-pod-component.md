---
layout: post
title: iOS项目组件化
date:   2018-09-20 22:06:05
categories: iOS
tags: [iOS项目组件化]
---

## 0. 组件化场景
适用于同时维护多个APP的场景，一般通过cocoapod管理，常称pod库。

## 1. 组件化前提
需要组件化的模块已解偶，即能独立或依赖其他组件或函数库编译通过。

## 2. 组件划分粒度
从业务相关度来划分可分为业务逻辑无关的组件、业务逻辑组件。前期可以将业务逻辑无关模块组件化。业务逻辑组件划分的需综合考虑各APP的业务场景共用的粒度。

## 3. 创建组件的步骤
a. 创建组件版本仓库，本文简称versionGit，假设地址为http://192.168.0.34/iOSGroup/iOSRepo.git

b. 关联本地版本库和组件远程版本库。
```
$pod repo add iOSRepo http://192.168.0.34/iOSGroup/iOSRepo.git
```
c. 生成pod库(含测试项目）
```
pod lib create podname
```
一路的选择输入后生成一个名称为podname的目录，其中podname.podspec文件 是一段ruby脚本实现的pod描述文件。下面是生产环境中的一个的名为TXLog.podspec的pod描述文件，实现根据环境变量podname_lib或all_lib设置组件通过源码还是framework引入到项目中。看不懂的先去补一下ruby基础知识吧。

podname.podspec
```
Pod::Spec.new do |s|
  s.name         = "podname"
  s.version      = "1.0.0"
  s.summary      = "Log tools, for term print and local file log or remote log"
  s.description  = <<-DESC
                    Log tools, for term print and local file log or remote log. Encapsulate from CocoaLumberjack
                   DESC
  s.homepage     = "http://www.icodingprogram.com"
  s.license = { :type => 'MIT', :text => <<-LICENSE
            Copyright up366 2018-2020
                       LICENSE
                              }
  s.author       = { "9drops" => "zhanbz@gmail.com" }
  s.source       = { :git => "http://192.168.0.34/iOSGroup/#{s.name}.git", :tag => "#{s.version}" }
  s.platform     = :ios, "8.0"
  s.requires_arc = true
  
   s.resource_bundles = {
      "#{s.name}" => ["#{s.name}/Assets/*.{png,xib,plist}"]
  }
  
  #s.module_map   = "#{s.name}/Classes/CommonCrypto/module.modulemap" 
  #如果自己已经打包好bundle
  #s.resources = "#{s.name}/Assets/*.bundle"
  s.dependency "CocoaLumberjack", "~> 2.0.0"
  
  if ENV["#{s.name}_lib"] || ENV["all_lib"]
    s.source_files = "#{s.name}/Classes/#{s.name}.h" , "#{s.name}/Classes/**/*.h"
    s.vendored_frameworks = "#{s.name}/Products/#{s.name}.framework"
    s.prepare_command = "/bin/sh  build_framework.sh #{s.name}"
  else
    s.source_files = "#{s.name}/Classes/*.{m,h,c}", "#{s.name}/Classes/**/*.{m,h,c}", "#{s.name}/Classes/**/**/*.{m,h,c}"
    s.public_header_files = "#{s.name}/Classes/#{s.name}.h", "#{s.name}/Classes/**/*.h"
  end
  
end
```

将需要组件化的源码和资源文件分别放到podname/podname下的Classes和Assets目录下。资源bundle包在framework中时，需要用NSBundle bundleForClass:方法获取到bundle里的资源文件，形如：
```
NSBundle *bundle = [NSBundle bundleForClass:[podname class]];
NSString *filePath = [bundle pathForResource:@"PlatformNames" ofType:@"plist" inDirectory:@"podname.bundle"];
```

d. 检查组件是否可用

* 只检查本地代码
```
pod lib lint --sources='组件版本库地址,https://github.com/CocoaPods/Specs'
```
* 检查远程组件库代码
```
pod spec lint --sources='组件版本库地址,https://github.com/CocoaPods/Specs'
```

e. push本地pod库到组件远程仓库

假设组件远程仓库已创建，简称podGit, 地址为：http://192.168.0.34/iOSGroup/podname.git

用到的命令：
```
$ git add -A && git commit -m "你的更新说明" 
$ git tag 1.0.0 #tag值和.podspce里的version相同，并且必须设置
$ git push --tags  
$ git push origin master  

$ git tag -d 0.0.68 #删除本地tag
$ git push origin :refs/tags/0.0.68 #删除远程库tag
```
f. 上传组件描述文件到组件版本库(http://192.168.0.34/iOSGroup/iOSRepo.git)
```
pod repo push iOSRepo podname.podspec --allow-warnings --verbose --sources=http://192.168.0.34/iOSGroup/iOSRepo.git,https://github.com/CocoaPods/Specs.git
```

g. 生成framework
通过pod package命令生成framework动态库,并将此动态库移动到podname.podspec中s.vendored_frameworks值对应的路径。并将此动态库上传到podGit。
```
pod package podname.podspec --dynamic  --spec-sources=http://192.168.0.34/iOSGroup/iOSRepo.git,https://github.com/CocoaPods/Specs.git   --force  --verbose
```

## 4. 注意
* 组件版本仓库只存放各版本对应的后缀名为podspec的描述文件
* 组件仓库存放对应的源码、资源文件等

## 5. 常见的错误

* 报错 ··· error: include of non-modular header inside framework module [-Werror,-Wnon-modular-include-in-framework-module]
解决办法1：如果pod编译为静态库，在pod lib lint 或者 pod spec lint 以及 pod repo push 时候加上   --use-libraries选项。当然，在提交的时候也要加上，例如，pod repo push <repoName> <podspec> --use-libraries

解决办法2:因为必须用动态库，所以不能用--use-libraries 选项，问题出在.h文件import了其他pod或framework的头文件，必须将被引用的头文件放到.m里或者将头文件放到modulemap里，具体格式如下，同时podspec也需要增加module_map字段，详见上面的podname.podspec.

Classes/CommonCrypto/ 目录下包含如下2个文件：

CommonCryptoHeader.h  
```
#import <CommonCrypto/CommonCrypto.h>
#import <CommonCrypto/CommonHMAC.h>
```
module.modulemap
```
module CommonCrypto [system] {
    header "CommonCryptoHeader.h"
    export *
}
framework module Foundation_TX {
}
```


* pod lib lint 报错
    ERROR | [iOS] public_header_files: The pattern includes header files that are not listed in source_files (podname/podname/Classes/podname.h).
导致的原因为： s.source_files = "TXLog/Classes/*.{m, h, c}" , {}中h,和c中间有空格导致的。


* pod lib lint和pod repo push都成功，但是pod package报错
  可能的原因：依赖的私有pod更新后没有上传到版本库

* 已经添加了私有库但搜不到的解决方案：
rm ~/Library/Caches/CocoaPods/search_index.json,再用reop search pod名字

## 6. 简化pod创建过程的脚本

* get_pod_version_in_podspec.sh - 获取 .podspec中version的值（git tag版本），供其他脚本使用 

* pod_lib_lint.sh - 封装pod lib lint命令， 此脚本使用本地源码编译目标文件

* pod_repo_push.sh - 封装pod repo push命令，推送本地podspec到repo版本库，此脚本使用git repository上对应tag版本源码编译目标文件 

* build_framework.sh - 封装pod package --dynamic 命令, 此脚本使用git repository上对应tag版本源码生成framework动态库，此脚本执行前后都需调用push_to_master.sh 命令将本地源码上传到git repository 

* push_to_master.sh - 推送.podspec中version的值（git tag版本）git repository，此脚本有一个必选参数message:git 提交消息

包含脚本的pod模版见[podTemplate](https://github.com/9drops/podTemplate)

**使用此模版创建自己的pod：**
1. 用create_pod.sh脚本创建自己的podname。
2. 拷贝podTemplate/下所有shell脚本到podname目录下，将podTemplate.podspec重命名为podname.podspec。
3. 修改podname.podspec满足实际需求。
4. 将用于创建pod的源码、资源文件分别放到podname/Classes和podname/Assets下, 生成的framework路径设定为podname/Products。
5. 执行pod_lib_lint.sh脚本，如果成功，执行pod_repo_push.sh脚本,失败则解决问题。
6. 执行build_framework.sh脚本，如果成功，执行pod_repo_push.sh和pod_repo_push.sh，否则检查git repository上源文件或依赖的私有pod是否和本地一致，可用的操作有清除自己和依赖pod的本地缓存。

## 7. pod组件以源码、framework形式引入项目时的切换

**相关描述**
* 前期组件不太稳定，需要经常调试代码，建议用pod install 默认安装源码。

* 后期组件稳定后，为了减少编译的时间，建议用all_lib=1 pod install安装pod，组件通过framework动态库引入到项目中
其中all_lib为环境变量，值为1则所有的组件通过framework 动态库的形式引入到项目中，
PODNAME1_lib=1 PODNAME2_lib=1 pod install,  则仅名字为PODNAME1、PODNAME2... PODNAMEn的组件通过framework 动态库的形式引入到项目中。


**源码和framework二进制切换前的步骤**
* 删除*.xcworkspace。
* 清除pod缓存，$pod cache clean PODNAME。
* 删除项目下Pods/PODNAME目录（此操作可以通过执行clean_pods.sh脚本实现，其中clean.list为待清除的组件列表，因为pod cache clean PODNAME无法清除所有跟PODNAME有关的缓存，此脚本可能需要执行多次）。


