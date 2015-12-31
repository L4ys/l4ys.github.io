---
layout: post
title: Dump iOS app headers
category: others
---

有時候有些特殊需求，需要 dump 出 iOS App 的 headers


一般的 iOS app 檔案都是被加密過的狀態，必須先執行再從記憶體中 dump 出解密過後的內容

可透過 Stefan Esser 的 [dumpdecrypted](https//github.com/stefanesser/dumpdecrypted) 來進行

<!--more-->


clone 下來 make 好之後，將編譯完成的 dumpdecrypted.dylib 上傳到 iPhone

`$ scp ./dumpdecrypted.dylib root@192.168.2.250:`

 ssh 上去
`$ ssh root@192.168.2.250`

注意以下指令皆在 iPhone 上執行

執行要 dump 的 app 並找出目標 app 的檔案路徑，以 Facebook 為例子:

```sh
$ ps aux | grep Facebook.app
mobile    1010   0.0  9.8   430824  50560   ??  Ss    3:53PM   0:22.49 /var/mobile/Applications/F81504AA-06C3-4CAD-ADED-D97EE5726163/Facebook.app/Facebook
root      1047   0.0  0.1   264836    324 s000  S+    3:56PM   0:00.01 grep Facebook.app
```

執行檔路徑是
`/var/mobile/Applications/F81504AA-06C3-4CAD-ADED-D97EE5726163/Facebook.app/Facebook`

Inject dylib:

```sh
$ DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Applications/F81504AA-06C3-4CAD-ADED-D97EE5726163/Facebook.app/Facebook mach-o decryption dumper
```

會出現像下面的訊息文字

```
DISCLAIMER: This tool is only meant for security research purposes, not for application crackers.

[+] offset to cryptid found: @0x8aa78(from 0x8a000) = a78
[+] Found encrypted data at address 00004000 of length 20873216 bytes - type 1.
[+] Opening /private/var/mobile/Applications/F81504AA-06C3-4CAD-ADED-D97EE5726163/Facebook.app/Facebook for reading.
[+] Reading header
[+] Detecting header type
[+] Executable is a plain MACH-O image
[+] Opening Facebook.decrypted for writing.
[+] Copying the not encrypted start of the file
[+] Dumping the decrypted data into the file
[+] Copying the not encrypted remainder of the file
[+] Setting the LC_ENCRYPTION_INFO->cryptid to 0 at offset a78
[+] Closing original file
[+] Closing dump file
```

dump 的結果會存放在該 app 的 tmp 目錄下

---

decrypt 完成之後就可以用 class-dump 來 dump 出 header:

```sh
$ class-dump -H Facebook.decrypted -o FBHeaders
```

