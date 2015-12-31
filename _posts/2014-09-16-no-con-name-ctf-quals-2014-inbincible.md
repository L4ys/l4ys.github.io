---
layout: post
title: 'No cON Name CTF Quals 2014: inBINcible'
category: writeup
---

> Get the key. The flag is: "NcN_" + sha1sum(key)
> https://ctf.noconname.org/chdownloads/inbincible
> Points: 400

題目為 ELF 32bit ELF，執行後只會顯示 `Nope!` 訊息
從程式中的函數名稱 `_rt0_go()` 可以看出是 [Go](http://golang.org/) 的執行檔

<!--more-->

分析 `runtime_main()` 後發現重點在於 `test()` 函數中，
會判斷 argv[1] 長度為 16 bytes ，接著於迴圈中呼叫 `runtime_newproc()` 執行 16次 main_func_001()</code>，
再透過 `runtime_chanrecv1()` 取得結果。

由於 `test()` 每次只透過 `main_func_001()` 處理一個字元，並回傳該字元是否正確，
因此我們可以在關鍵處設下斷點，再根據程式是否有執行到斷點，以每次一個 char 的方式爆出密碼

`0x0804911D` 是個關鍵跳轉，在 `0x0804911F` 上下一個斷點可以判斷 char 是否正確

```
.text:0804910F                 call    runtime_chanrecv1
.text:08049114                 mov     edi, [esp+0C0h+var_A0]
.text:08049118                 cmp     [esp+0C0h+var_A5], 0
.text:0804911D                 jz      short loc_8049127
.text:0804911F                 mov     ecx, 1
```

測試輸入 AAAAAAAAAAAAAAAA BBBBBBBBBBBBBBBB ...  會發現當輸入為 GGGGGGGGGGGGGGGG 時斷點會被觸發一次
嘗試猜測第二個字元，輸入 G0GGGGGGGGGGGGGG 時斷點會被觸發兩次，表示前兩位 key 為 G0w1n

接著就開始透過 gdb script 搭配 [peda](https://github.com/longld/peda) 窮舉出 key: 

```py
#!/usr/bin/env python
import sys
import string

list = string.letters
list += string.digits
list += string.punctuation

peda.execute("file ./inbincible")
peda.execute("br *0x0804911F")

key = ""
for i in range(16):
    for c in list:
        guess_key = key + str(c) + "A" * (15 - i)  
        sys.stderr.write("Testing %s...\n" % guess_key)   
        peda.execute("r " + guess_key)

        break_count = 0
        for j in range(i + 1):            # Testing key[0] needs to hit the bp for 1 time, key[1] for 2 ...
            if peda.getpid() is not None: # still running?
                peda.execute("c")
                break_count += 1

        if break_count == i + 1:
            key += c
            break

sys.stderr.write("Done!!!\n")
sys.stderr.write("The key is:\n%s\n" % key)

peda.execute("q")
```

```sh
$ peda -x ./crack.py > /dev/null
Testing aAAAAAAAAAAAAAAA...
Testing bAAAAAAAAAAAAAAA...
Testing cAAAAAAAAAAAAAAA...
Testing dAAAAAAAAAAAAAAA...
Testing eAAAAAAAAAAAAAAA...
...
Testing G0w1n!C0ngr4t5!5...
Testing G0w1n!C0ngr4t5!6...
Testing G0w1n!C0ngr4t5!7...
Testing G0w1n!C0ngr4t5!8...
Testing G0w1n!C0ngr4t5!9...
Testing G0w1n!C0ngr4t5!!...
Done!!!
The key is:
G0w1n!C0ngr4t5!!</code></pre>
