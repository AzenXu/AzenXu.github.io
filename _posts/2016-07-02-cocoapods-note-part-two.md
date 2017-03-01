---
layout: post
title: Cocoa pods学习笔记（二）Podsepc文件 & 提交到CocoaPods
---
>此系列文章均整理、精简自[pluto-y大神](http://www.pluto-y.com/cocoapods-getting-stared/)的博客，感谢大神~

##一、库分类  

	公有库：公开给所有人的库
	私有库：不公开给所有人的库  
##二、Podsepc文件解析  

	1. 不管是做公有库或者是私有库都是必须配置这个文件
	2. 这个文件是告诉Cocoapods库的基本信息，包括：
		a. 版本号
		b. 获取的地址
		c. 包含哪些文件
		d. Demo：
			i. Pod::Spec.new do |s|  
                s.name         = "iOS-Echarts"
                s.version      = "1.1.4"
                s.summary      = "A custom component for the ecomfe's echarts."
                s.homepage     = "https://github.com/Pluto-Y/iOS-Echarts"
                s.license      = { :type => "MIT", :file => 'LICENSE.md' }
                s.author       = { "PlutoY" => "kuaileainits@163.com" }
                s.platform     = :ios, "7.0"
                s.source       = { :git => "https://github.com/Pluto-Y/iOS-Echarts.git", :tag => s.version}
                s.frameworks   = 'UIKit'
                s.source_files = "iOS-Echarts/**/*.{h,m}"
                s.resource     = "iOS-Echarts/Resources/**"
                s.requires_arc = true
                end  
	3. 新建或者需要新提交一个版本时，需更新此文件
	4. 解析：
		a. source_file：哪些目录可被包含
		b. resource：哪些文件作为资源文件引入（可用匹配符：*, **, ?, [set], {p,q}, \）
		c. 其余可用属性：sourcefiles, publicheaderfiles,privateheaderfiles,vendoredframeworks,vendoredlibraries,resourcebundles,resources,excludefiles,preservepaths,module_map.
	5. do |名字|（作用：起别名）：
		a. 场景
			i. Pod::Spec.new do |s|
			ii. s.subspec 'SubModule' do |sm|
	6. Subspec
		a. Demo：
			i. s.subspec 'Security' do |ss|  
                ss.source_files = 'AFNetworking/AFSecurityPolicy.{h,m}'
                ss.public_header_files = 'AFNetworking/AFSecurityPolicy.h'
                ss.frameworks = 'Security'
                end  
			ii. 声明一个子模块：父节点名.subspec '子模块名' do |子节点名|
			iii. 子模块里面一般都只需要配置一些文件匹配的信息、资源文件的信息以及一些依赖的信息即可
	7. 其他配置
		a. Cocoapods的Podspec语法

##三、提交公有库到CocoaPods  

	1. 申请一个账号：
		a. 语法为pod trunk register 邮箱 '昵称' --description='设备信息'
			i. 通过pod trunk me来查看是否"注册"成功
		b. 在你项目的根目录下运行：pod spec create '项目名'  
			i. Cocoapods产生一个包含一部分基础内容的Podspec文件
		c. pod lib create '项目名'，创建一个Cocoapdos定义好的一个工程模板
			i. 通过这个命令Cocoapods会给你创建好,Podspec文件，travis文件,MIT Lisence文件以及README.md文件等。
	2. 检查Podspec文件
		a. pod lib lint Name.podspec
		b. 参数：
			i. --verbose
			ii.  * --allow-warnings 可以屏蔽警告 
			iii. * --fail-fast 出现第一个错误的时候就停止
			iv. * --use-libraries 如果用到的第三方中需要使用库文件的话，则会用到这个参数。（如在项目中依赖了ShareSDK的情况） 
			v. * --sources 如果一个库的podspec包含了除了Cocoapods仓库以外的其他库的引用，则需要该参数指明，用逗号分隔，这个参数对于私有库是非常有用的
			vi. * --subspec=NAME 检验某一个子模块的情况
	3. 上传项目
		a. pod trunk push 项目名.podspec   - 默认我们的Podspec文件提交到Cocoapod的仓库(Specs)
		b. 新的tag版本也通过该命令上传
		c. pod search 名字查询是否上传成功。
	4. 将合作者拉入项目pod trunk add-owner '项目名' '邮箱'
