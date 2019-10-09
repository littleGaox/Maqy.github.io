---
layout:     post
title:      CocoaPods笔记
subtitle:   
date:       2999-12-31
author:     Maqy
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - CocoaPods
---



官网网址 https://guides.cocoapods.org/

pod init  创建podfile
pod install 安装
pod outdated 查看有更新的版本的pod 
pod update PODName 更新某个pod



podfile引入组件的不同方式：

```
pod 'Alamofire', :path => '~/Documents/Alamofire'
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git'
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git', :branch => 'dev'
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git', :tag => '3.1.1'
pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git', :commit => '0f506b1c45'
```



cocoapods实际做的事情参考https://guides.cocoapods.org/using/using-cocoapods.html

cocoapods问题及解决方案参考https://guides.cocoapods.org/using/troubleshooting.html



#安装ruby 
gem install ruby 

#升级ruby 
sudo gem update --system

#检查版本
ruby -v



更新rubu镜像

gem sources --remove https://rubygems.org/
gem sources -a https://ruby.taobao.org/
#检查镜像是否更新成功
gem sources -l

