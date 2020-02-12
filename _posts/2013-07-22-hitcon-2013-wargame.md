---
layout: post
title: HITCON 2013 Wargame 心得
category: others
---

第一次參加 HITCON ，記錄一下解Wargame的過程
蠻好運的，還可以拿到第10名，不過如果早點上傳KEY就能領錢了...Orz
一個人解真的有點吃力啊 囧 

![score](http://4.bp.blogspot.com/-mNiuzomMr3Q/Utkv0rRR2MI/AAAAAAAAAOA/Ndj1mZ715A8/s1600/1005289_496004637141738_1246511821_n.jpg)

<!--more-->

--- 

## Pwn 1
> ____ the planet !

一開始就先找這題XD 萬年老梗 hack the planet !

## Web 1
> There are something strange in WARGAME site.
> Find the key!

繞一繞，看了一下 konami.js
不小心看到這段
![konami](http://3.bp.blogspot.com/-sS7lXuAZdLU/Uex8Jm4wIJI/AAAAAAAAAHk/BS-VlwG7k50/s1600/1.png)
把iPhone拿出來滑幾下 上上下下左右左右點點點 ... 就噴key了...囧! 

![key](http://4.bp.blogspot.com/-brikaMpXhj8/Uex-w3Na-FI/AAAAAAAAAH0/j3XwWFpwCr8/s1600/photo.jpg)
用電腦的瀏覽器的話看起來是要輸入 上上下下左右左右 B A ENTER

## Binary 1
> Give me 5000 point !!

是個小遊戲，不囉嗦 直接修改分數限制
結果...
![](http://4.bp.blogspot.com/-Z_-imI5C57c/UeyAjcwmp2I/AAAAAAAAAIE/7NvdIRnewJg/s1600/2.png)

開OD稍微看了一下發現是Python寫的，直接用7z打開..
![7z](http://3.bp.blogspot.com/-nD7w7fnsRiQ/UeyAjaF7k8I/AAAAAAAAAII/tVd2w5SnHc0/s1600/3.png)

## Forensics 1

一個加密的zip，照壓縮檔內容看來，zip 內的 hero.png 跟 hitcon 官網上的為同一張
找工具來做known-plaintext attack
  
## Web 2
> http://c.hackit.tw is a simple php guestboard,
> try to read http://c.hackit.tw/key.php

看一下php的src ，想辦法讓pos做點偏移，讀取到key.php個這字串就行了

pos塞1.75，會變成(1.75-1) * 128 + 32 = 128，就可以讀到第二篇的標題
把第二篇的標題塞 `"key.php"` 就搞定了

## Web 3
> http://d.hackit.tw is a simple member system,
> try to login as admin to read key !

要求輸入驗證碼，其實就藏在favicon裡面
![favicon](http://2.bp.blogspot.com/-OPKvIHcxXJc/UeyExlCt8mI/AAAAAAAAAIc/sBsxnIYFJ3M/s1600/3.png)

Get 網頁時順便把 favicon 弄下來放大就看得見了
至於帳號密碼我直接猜admin/admin，結果就進去了...
![login](http://2.bp.blogspot.com/-46Hwc-d6qwI/UeyEx7it-6I/AAAAAAAAAIg/s2MI9S9EfPc/s1600/4.png)

登入後又是一樣的把戲，輸入完就直接get key了

## Roulette 5.
> To be or not to be that is the question.

一個rar，開啟之後...還是一個rar，再開，又一個
![rar](http://2.bp.blogspot.com/-dCuwXpIffBU/UeyGL3-HH9I/AAAAAAAAAI0/Bi4NsJuayGk/s1600/5.png)

照編號來看大概是有100個... 用python寫了個腳本來解壓縮
結果....
![](http://1.bp.blogspot.com/-a7ix0AfXl0o/UeyGdhYPlPI/AAAAAAAAAJA/zrJcYxbAB9Q/s1600/5.png)

4D.rar 打開，裡面是個 5A.rar
大概猜得到是怎麼回事了吧...

開始寫python code，解壓縮並記錄檔名
可能臨時寫的code太爛了...跑了大概10分鐘才跑完

最後會得到一個food.bin，然後我們把這些BYTE組成一個執行檔，執行後...
![](http://3.bp.blogspot.com/-mWIQsww8TnE/UeyHobvFMFI/AAAAAAAAAJM/O58JTdu7c0U/s1600/6.png)
好，直接進IDA

<div class="separator" style="clear: both; text-align: center;"><a href="http://3.bp.blogspot.com/-B62DfE4MO-4/UeyIOaS_eCI/AAAAAAAAAJU/v-nynhTD96Y/s1600/7.png" imageanchor="1" style="margin-left: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/-B62DfE4MO-4/UeyIOaS_eCI/AAAAAAAAAJU/v-nynhTD96Y/s1600/7.png" /></a></div>
繞一繞 整理一下 關鍵部分大概是這樣 

```c
scanf( "%s", input );
pass = atoi( input ) ;

if ( ( 0x9A391B58 ^ ( ( pass << 17 ) | ( pass >> 15 ) ) ) == 0x795E5F5E )
  // 解key
```

那就寫個code來找能解密的key吧
噹噹~跑出來的結果是: -1576832589

<div class="separator" style="clear: both; text-align: center;"><a href="http://3.bp.blogspot.com/-3dbkH4FsLBg/UeyKkE7qeQI/AAAAAAAAAJk/ZParyXxsAxc/s1600/8.png" imageanchor="1" style="margin-left: 1em; margin-right: 1em;"><img border="0" src="http://3.bp.blogspot.com/-3dbkH4FsLBg/UeyKkE7qeQI/AAAAAAAAAJk/ZParyXxsAxc/s1600/8.png" /></a></div>


## Forensics 5.
> fix Rar header

這題有提示後根本送分XD
啊就 手動修一下rar的header

會解出一個文字檔，裡面有一串md5，丟Google 得 Key 
500分!

--- 

幾題感覺有點可惜的...

Forensics 2.. 都解得差不多了
大概是給了假的API Key，應該直接開C# 寫的

Binary2看了很久還是猜不到梗... 

在Pwn3 浪費了蠻多時間 都快解出來了... 一直鬼打牆
結果聽說那台機器有ASLR...-_-

感覺今年的Web Security不難 但我實在不擅長 囧
看到 apk 跟 osx直接放棄了..

如果windows題多一點就好了XD ( ...我到底帶macbook去幹嘛的 )

---

繼續加油吧!
