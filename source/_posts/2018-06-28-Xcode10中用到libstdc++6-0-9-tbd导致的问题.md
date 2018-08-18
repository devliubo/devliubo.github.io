---
layout: post
title: Xcode10中用到libstdc++6.0.9.tbd导致的问题
comments: true
author: devliubo
categories: iOS
date: 2018-06-28 13:47:02
tags: ["iOS","Xcode10","cocoapods"]
---

最近Apple发布了Xcode10的beta版本，其中一个变化是去掉了std相关的tbd（可以参考Xcode10的Release Node），Apple给出的原因是std库比较旧了，建议使用新版本替换，比如用libc++.tbd替换libstdc++6.0.9.tbd。这就导致之前依赖libstdc++6.0.9.tbd的工程，在升级到Xcode10后出现编译错误。

<!-- more -->

错误的具体信息如下图所示：

![ErrorInfo0](/images/2018-06-28-Xcode10中用到libstdc++6-0-9-tbd导致的问题/ErrorInfo0.png)

出现这个问题的原因是因为在开发中使用了第三方库，并且第三方库的podspec中libraries指定了stdc++6.0.9，cocoapods在安装依赖过程中，会在指定target下的other link flag中加入 -l"stdc++6.0.9" 导致编译不能通过。

那么要如何解决呢？这个问题可以通过使用cocoapods的post_install hooks来解决，解决的方法是在podfile中加入下面的代码，去掉所有pod对stdc++6.0.9的依赖：

```
#该方法会移除 所有pod 对stdc++.6.0.9库的依赖，建议仅在Xcode10上使用
post_install do |installer|
    installer.pods_project.targets.each do |target|
        target.build_configurations.each do |config|
            puts config.build_settings
            xcconfig_path = config.base_configuration_reference.real_path
            build_settings = Hash[*File.read(xcconfig_path).lines.map{|x| x.split(/\s*=\s*/, 2)}.flatten]
            build_settings['OTHER_LDFLAGS'][' -l"stdc++.6.0.9"'] = ''
            File.open(xcconfig_path, "w") do |file|
                build_settings.each do |key,value|
                    file.puts "#{key} = #{value}"
                end
            end
        end
    end
end
```

Ps:
这段hooks的原理是：找到所有的target的Pods-TargetName.debug.xcconfig和Pods-TargetName.release.xcconfig这两个文件（在Pod/Targets Support Files/TargetName目录下），然后将其中的OTHER_LDFLAGS字段中的-l"stdc++.6.0.9"去掉。
