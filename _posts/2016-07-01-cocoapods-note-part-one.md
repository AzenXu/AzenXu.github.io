---
layout: post
title: Cocoa pods学习笔记（一）Podfile文件剖析 & Pod命令解析
---
>此系列文章均整理、精简自[pluto-y大神](http://www.pluto-y.com/cocoapods-getting-stared/)的博客，感谢大神~
经过最近一段时间的模块化开发实践，组里决定从Carthage转到CocoaPods了。相比于Carthage，CocoaPods对共有组件的版本控制更好，打包的时候速度也更快些。

Cocoapods是一个框架依赖管理的一个管理工具，主要是用来管理框架一些开源库在项目中的引用。简而言之就是用来管理你的项目中对开源框架或自己公司子模块的依赖。
参考：
	1. [Cocoapods官网](https://cocoapods.org/)
	2. [Cocoapods Guide](https://guides.cocoapods.org/syntax/podfile.html#podfile)
	
##Podfile文件剖析
Podfile.lock文件：  

	1.保护pod本地文件们。
	2.在没有执行pod update命令的情况下，是不会讲已有的第三方依赖库进行升级的。所以运行pod install的情况下还是能编译通过的。  
	
pod：  

	1. 关于pod的使用在上面可以看得出来是pod '框架名' 参数。 
		a. pod '框架名'是固定的
		b. 基础参数
			i. 参数一： 版本号 可以是'> 3.7', '>= 3.7', '< 3.7', '3.7'以及'~> 3.7'（~>指的是正对最后一位，如使用'~> 3.7.4',意味着'>= 3.7.4'并且'< 3.8.0'的意思）
			ii. 参数二：地址 Cocoapods可以指定某一个git的目录或者是本地的目录（直接接上:git => 'https://github.com/gowalla/AFNetworking.git'。  表示一直用最新版本） 
		c. 私有库参数
			i. path参数：:path => '~/Documents/AFNetworking' （开发阶段，以开发模式进行其他库的引用，如：主工程引用子库）
			ii. 参数三：:branch => 'branch名'、:tag => 'tag名'、:commit => '提交号'。 （引用特定tag、branch、comit的内容）
			iii. 参数四：inhibitallwarnings! （用来避免那些第三方框架中带来的warnings）
	2. platform，依赖的库希望在哪个平台被编译（platform :ios, '7.0'。说希望采用iOS7.0的进行编译）
	3. target
		a. 指定所适配的target（Xcode中的target）
		b. 如果对于一些项目中你的不同target引用的框架不同的话，可以采用这个进行区分。
		c. target 'ScenicHotelDemo' do
		  # Comment this line if you're not using Swift and don't want to use dynamic frameworks
		  use_frameworks!
		
		  # Pods for ScenicHotelDemo
		  pod 'JFoundation'
		
		end
		
		target 'SceniHotelKit' do
		  # Comment this line if you're not using Swift and don't want to use dynamic frameworks
		  use_frameworks!
		
		  # Pods for SceniHotelKit
		  pod 'JFoundation'
		
		end
	4. use_frameworks!
		a. 强制把所有项目编译成动态库。（Swift必须）
	5. source
		a. 这个参数是指Cocoapods从哪些仓库(Spec)中获得框架的源代码，（同时使用开源库以及自己私有库的情况下，这个参数很有用）
		b. 只需要在Podfile文件开头列出你需要引用库的所有仓库地址即可。
		c. #open source
		source 'https://github.com/Cocoapods/Specs.git'
		#JSpecs
		source 'http://192.168.1.111/iOS/Specs.git'
	6. 收尾
		a. Cocoapods Guide
		b. platform :ios, '8.0'
		#open source
		source 'https://github.com/Cocoapods/Specs.git'
		#JSpecs
		source 'http://192.168.1.111/iOS/Specs.git'
		
		target 'ScenicHotelDemo' do
		  # Comment this line if you're not using Swift and don't want to use dynamic frameworks
		  use_frameworks!
		
		  # Pods for ScenicHotelDemo
		  pod 'JFoundation'
		
		end
		
		target 'SceniHotelKit' do
		  # Comment this line if you're not using Swift and don't want to use dynamic frameworks
		  use_frameworks!
		
		  # Pods for SceniHotelKit
		  pod 'JFoundation'
		
		end
		
##pod install 和 pod update  

	1. 用途：更新或者是安装一个新的第三方框架
		a. pod install：是在有新的第三方框架引入是运行
		b. pod update：是纯粹为了更新本地的第三框架
		c. pod repo update：更新本地已有的所有第三框框架
	2. 参数
		a. --no-repo-update
			i. 在执行pod install和pod update两条命令时，会执行pod repo update的操作
			ii. 这个命令可以只更新当前项目的第三方框架
		b. --verbose 和 --silent
			i. 作用：控制pod命令执行过程中的输出
			ii. --silent不展示输出的情况
            iii. --verbose展示具体的出错信息（啰嗦模式）
