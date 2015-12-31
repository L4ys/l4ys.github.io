---
layout: post
title: 在 iOS 模擬器上測試 mobilesubstrate tweak
category: others
---

前幾天才發現可以在 iOS 模擬器上執行 mobilesubstrate 的 tweak

參考 [這篇文章](http://sharedinstance.net/2013/10/running-tweaks-in-simulator/) 寫了些記錄

<!--more-->
不過似乎只限於 hook SpringBoard 的 tweak ，其他部分還沒有測試

![hook](http://3.bp.blogspot.com/-20NmPuIuHzs/UtjweZJSDRI/AAAAAAAAANw/v8h_eR-TGf8/s1600/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7+2014-01-17+%E4%B8%8B%E5%8D%884.56.46+1.png)

---

步驟如下

```sh
# 下載 MobleSubstrate
$ curl -OL http://apt.saurik.com/debs/mobilesubstrate_0.9.4001_iphoneos-arm.deb

# 解壓縮然後在本機配置 library
$ dpkg-deb -x mobilesubstrate_0.9.4001_iphoneos-arm.deb substrate
$ sudo mv substrate/Library/Frameworks/CydiaSubstrate.framework /Library/Frameworks
$ sudo mv substrate/Library/MobileSubstrate /Library/MobileSubstrate
$ sudo mv substrate/usr/lib/* /usr/lib
```

接著要修改模擬器來於執行 SpringBoard 時 inject tweak
由於 iOS7 之後的模擬器是由 launchd_sim 來啟動，因此舊有的方法無法使用
需要修改 SDK 中 LaunchDaemon 的 plist 達成

先將 

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator7.0.sdk/System/Library/LaunchDaemons/com.apple.SpringBoard.plist
```

備份到別處 ( 放在這目錄會造成 SpringBoard 被執行兩次 )

編輯 com.apple.SpringBoard.plist，新增一個名為 `EnvironmentVariables` 的 Dictionary
並在 EnvironmentVariables 下新增一個 Key 名為 `DYLD_INSERT_LIBRARIES`
內容為要 inject 的 tweak 的路徑 ( 不是 MobileSubstrate.dylib )

例如 `/Library/MobileSubstrate/DynamicLibraries/SpotLock7.dylib`: 
![dylib](http://1.bp.blogspot.com/-VzHYeek5sWs/UtsyNVxzSYI/AAAAAAAAAOQ/37AYMf8ijRI/s1600/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7+2014-01-19+%E4%B8%8A%E5%8D%8810.01.37.png)

---

再來需要修改 theos
開啟 `$THEOS/makefiles/targets/Darwin/simulator.mk`

找到這行

```sh
_TARGET_OSX_VERSION_FLAG = -mmacosx-version-min=$(if $(_TARGET_VERSION_GE_4_0), 10.6,10.5)`
```

取代為

```sh
_TARGET_VERSION_GE_7_0 = $(call __simplify,_TARGET_VERSION_GE_7_0,$(shell $(THEOS_BIN_PATH)/vercmp.pl $(_THEOS_TARGET_SDK_VERSION) ge 7.0))
_TARGET_OSX_VERSION_FLAG = $(if $(_TARGET_VERSION_GE_7_0),-miphoneos-version-min=7.0,-mmacosx-version-min=$(if $(_TARGET_VERSION_GE_4_0),10.6,10.5))
```

---

再來處理 linking 部分
libsubstrate.dylib 並沒有 x86 的 slice ，因此要下載另個libsubstrate.dylib來取代它

```sh
$ curl -O http://cdn.hbang.ws/dl/libsubstrate_arm64.dylib
$ mv $THEOS/lib/libsubstrate.dylib libsubstrate.dylib.bak
$ mv ./libsubstrate_arm64.dylib $THEOS/lib/libsubstrate.dylib
```

---

最後修改 tweak 的 makefile ，使其可以正確編譯出給 simulator 執行的 dylib

加上

```sh
export IPHONE_SIMULATOR_ROOT=/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator7.0.SDK
```

然後指定 TARGET 為 simulator:
`TARGET = simulator`

執行 make install 之後 tweak 就會被安裝到 simulator 上

若要重啟 SpringBoard 必須自己手動 killall -9 SpringBoard

