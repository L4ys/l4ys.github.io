---
layout: post
title: Boston Key Party CTF 2014 - VM
category: writeup
tags: [reverse]
---

> this vm needs a license to run. we don't have the license!
> http://bostonkeyparty.net/challenges/vm-2fbed3f5a894d56be6b2ba328f9e2411

這幾天有點時間，解了幾題去年 Boston Key Party CTF 的題目，做個紀錄

<!--more-->

```sh
$ file vm-2fbed3f5a894d56be6b2ba328f9e2411
vm-2fbed3f5a894d56be6b2ba328f9e2411: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, stripped
```

64bit ELF，丟進 IDA Pro 稍微看一下可以發現這是一支以解譯器執行 vm code 的程式

程式首先從 0x004018B0 讀取 912 bytes 的 vm opcode ，再透過位於 0x00400D70 的解譯函數執行

分析一下解譯函數， vm opcode 是被 encode 過的，取指令來執行前會先進行 Decode:

```c
int fetch_op(unsigned char *vm_op, int index)
{
  unsigned char n = 0;
  
  if ( index > 0 )
    n = vm_op[index - 1];
  return ((index - 11 * (((780903145 * index) >> 63) + (780903145 * index >> 33))) ^ n ^ vm_op[index]);
}
```

---

參考解譯函數自幹個簡單的 vm dissambler:

```py
#!/usr/bin/env python
# encoding: utf-8
import sys
import string
from struct import unpack,pack

def fetch_op(vm_op, index):
    p = 0
    if index > 0:
        p = ord(vm_op[index - 1])

    return ((index - 11 * (((780903145 * index) >> 63) + (780903145 * index >> 33))) ^ p ^ ord(vm_op[index]))

def unvm(vm_op):
    print "Unvm, len:", hex(len(vm_op))
    i = 0
    while i < len(vm_op):
        print "0x%08X:" % i,
        op = vm_op[i]
        i += 1

        if op == 0x2B: # add
            print "ADD [SP],", hex(vm_op[i])
            i += 1
        elif op == 0x2D: # sub
            print "SUB [SP],", hex(vm_op[i])
        elif op == 0x3D: # cmp
            print "CMP [SP],", hex(vm_op[i])
            i += 1
        elif op == 0x44: # decrypt
            _len = unpack("<i", pack("BBBB", *vm_op[i:i+4]))[0]
            i += 4
            print "VM_Decrypt", "len:", _len, "addr:", hex(i)
        elif op == 0x4E: # nop
            continue
        elif op == 0x4F: # fopen
            print "VM_fopen"
        elif op == 0x50: # push xx
            print "PUSH", hex(vm_op[i]),
            if chr(vm_op[i]) in string.ascii_letters + string.digits +  string.punctuation:
                print "(", chr(vm_op[i]),")"
            else:
                print ""
            i += 1
        elif op == 0x51: # pop
            print "POP"
        elif op == 0x52: # reset sp
            print "VM_Reset_Stack"
        elif op == 0x5F: # exit
            print "VM_Exit"
        elif op == 0x61: # xchg high nibble
            print "NXCHG [SP], [SP - %02X]" % vm_op[i]
            i += 1
        elif op == 0x69: # not
            print "NOT [SP]"
        elif op == 0x6a: # jmp
            print "JMP", hex(vm_op[i]+i+1)
            i += 1
        elif op == 0x6b: # je
            print "JE", hex(vm_op[i]+i+1)
            i += 1
        elif op == 0x6c: # jne
            print "JNE", hex(vm_op[i]+i+1)
            i += 1
        elif op == 0x67: # fgetc
            print "VM_fgetc"
        elif op == 0x6e: # push 0
            print "PUSH 0x00"
        elif op == 0x70:
            print "VM_fputc"
        elif op == 0x71: # nibble swap
            print "NSWAP [SP]"
        elif op == 0x72:
            print "VM_Encode [SP]"
        elif op == 0x73: # xchg
            print "XCHG", "[SP], [SP - %02X]" % vm_op[i]
            i += 1
        elif op == 0x74: # xor with 2 ^ n
            print "XOR", "[SP],", hex(1 << vm_op[i])
            i += 1
        elif op == 0x78: # xor
            print "XOR", "[SP],", hex(vm_op[i])
            i += 1
        else:
            print "Unknown op:", hex(op)

if __name__ == "__main__":
    # Read encoded op codes from file
    if len(sys.argv) == 2:
        f = open(sys.argv[1])
    else:
        f = open("./vm-2fbed3f5a894d56be6b2ba328f9e2411")
        f.seek(0x18B0)

    vm_op_encoded = f.read(912)
    f.close()

    # Decode op codes
    unvm([fetch_op(vm_op_encoded, i) for i in range(len(vm_op_encoded))])

```

總共包含 20 多個指令
值得注意的是其中包含一些開檔讀檔還有進行 AES Decrypt 的函數:

 - VM_fopen: 開檔
 - VM_fgetc: 從檔案讀 1byte 到 AES Key 中
 - VM_Decrypt: 以 AES-128 OFB Decrypt vm code

初始的 Key 跟 IV 都是 `000102030405060708090A0B0C0D0E0F`

執行一下得到反組譯結果:

```
Unvm, len: 0x390
0x00000000: PUSH 0x6c ( l )
0x00000002: PUSH 0x69 ( i )
0x00000004: PUSH 0x63 ( c )
0x00000006: PUSH 0x65 ( e )
0x00000008: PUSH 0x6e ( n )
0x0000000A: PUSH 0x73 ( s )
0x0000000C: PUSH 0x65 ( e )
0x0000000E: PUSH 0x2e ( . )
0x00000010: PUSH 0x64 ( d )
0x00000012: PUSH 0x72 ( r )
0x00000014: PUSH 0x6d ( m )
0x00000016: PUSH 0x00
0x00000017: PUSH 0xc
0x00000019: VM_fopen
0x0000001A: VM_Decrypt len: 881 addr: 0x1f
0x0000001F: Unknown op: 0x6d
0x00000020: Unknown op: 0xe6
0x00000021: Unknown op: 0x35
0x00000022: Unknown op: 0xd9
... ( 下略 )
```

可以看出程式會先開啟 license.drm，接著 Decrypt 從 +0x1F 開始的 vm opcode
為了取得 Decrypt 之後的 opcode ，寫個 gdb script 在 Decrypt 完成時 dump op code 出來: 

```sh
file ./vm-2fbed3f5a894d56be6b2ba328f9e2411
# Get address of vm opcode
br *0x401598 
r
set $vm_op = $rsi
del br

# Dump decrypted vm_opcode
br *0x400D67
c
dump binary memory dump $vm_op $vm_op+912
q
```

然後再餵給 Disassembler:

```
0x0000001F: VM_fgetc
0x00000020: XOR [SP], 0xaa
0x00000022: VM_fgetc
0x00000023: XOR [SP], 0xbb
0x00000025: VM_fgetc
0x00000026: XOR [SP], 0xcc
0x00000028: XCHG [SP], [SP - 01]
0x0000002A: VM_fgetc
0x0000002B: XOR [SP], 0xdd
0x0000002D: XCHG [SP], [SP - 03]
0x0000002F: CMP [SP], 0xf5
0x00000031: JNE 0x3e
0x00000033: CMP [SP], 0x9c
0x00000035: JNE 0x3f
0x00000037: CMP [SP], 0x47
0x00000039: JNE 0x3f
0x0000003B: CMP [SP], 0x1
0x0000003D: JE 0x52
0x0000003F: PUSH 0xa
0x00000041: PUSH 0x21 ( ! )
0x00000043: PUSH 0x65 ( e )
0x00000045: PUSH 0x70 ( p )
0x00000047: PUSH 0x6f ( o )
0x00000049: PUSH 0x4e ( N )
0x0000004B: VM_fputc
0x0000004C: VM_fputc
0x0000004D: VM_fputc
0x0000004E: VM_fputc
0x0000004F: VM_fputc
0x00000050: VM_fputc
0x00000051: VM_Exit
0x00000052: PUSH 0xa
0x00000054: PUSH 0x3f ( ? )
0x00000056: PUSH 0x74 ( t )
0x00000058: PUSH 0x69 ( i )
0x0000005A: PUSH 0x20
(略)
0x000000D6: VM_Decrypt len: 689 addr: 0xdb
0x000000DB: Unknown op: 0x94
0x000000DC: Unknown op: 0xfc
0x000000DD: Unknown op: 0x64
0x000000DE: Unknown op: 0x15
0x000000DF: Unknown op: 0x89
```

+0xDB 開始又再出現需要 Decrypt 的區塊，不過中間多了一些可以分析的 code 

反編譯後，程式流程大約是:

 - 從 license.drm 讀取 4 byte 並取代目前 key 的 前 4 byte
 - 檢查 AES Key，必須滿足下列條件，否則印 "Nope" 並結束:
 - key[0] ^ 0xAA == 0xF5
 - key[1] ^ 0xBB == 0x9C
 - key[2] ^ 0xCC == 0x47
 - key[3] ^ 0xDD == 0x01
 - 從上面的條件我們可以算出正確的 license.drm 前 4 byte 為 `0x5F, 0x9C, 0x47, 0x01`

把這 4 byte 寫進 license.drm，再用同樣的方式取得 vm opcode 餵給 Disassembler

```sh
$ python -c "open('license.drm', 'wb').write('\x5F\x27\x8B\xDC')"
```

```
0x000000DB: VM_fgetc
0x000000DC: NOT [SP]
0x000000DD: XOR [SP], 0xee
0x000000DF: VM_fgetc
0x000000E0: NOT [SP]
0x000000E1: XOR [SP], 0xdd
0x000000E3: VM_fgetc
0x000000E4: NOT [SP]
0x000000E5: XOR [SP], 0xcc
0x000000E7: VM_fgetc
0x000000E8: NOT [SP]
0x000000E9: XOR [SP], 0xaa
0x000000EB: XCHG [SP], [SP - 03]
0x000000ED: XCHG [SP], [SP - 02]
0x000000EF: XCHG [SP], [SP - 01]
0x000000F1: CMP [SP], 0x55
0x000000F3: JNE 0x100
0x000000F5: CMP [SP], 0xcc
0x000000F7: JNE 0x101
0x000000F9: CMP [SP], 0xf6
0x000000FB: JNE 0x101
0x000000FD: CMP [SP], 0x7
0x000000FF: JE 0x114
0x00000101: PUSH 0xa
0x00000103: PUSH 0x21 ( ! )
0x00000105: PUSH 0x65 ( e )
0x00000107: PUSH 0x70 ( p )
0x00000109: PUSH 0x6f ( o )
0x0000010B: PUSH 0x4e ( N )
0x0000010D: VM_fputc
0x0000010E: VM_fputc
0x0000010F: VM_fputc
0x00000110: VM_fputc
0x00000111: VM_fputc
0x00000112: VM_fputc
0x00000113: VM_Exit
0x00000114: PUSH 0xa
0x00000116: PUSH 0x21 ( ! )
0x00000118: PUSH 0x67 ( g )
0x0000011A: PUSH 0x6e ( n )
0x0000011C: PUSH 0x69 ( i )
( 略 )
0x00000171: VM_Decrypt len: 529 addr: 0x176
0x00000176: Unknown op: 0xf2
0x00000177: VM_Reset_Stack
0x00000178: Unknown op: 0xc8
0x00000179: VM_fopen
0x0000017A: Unknown op: 0xbc
( 下略 )
```

跟上一個區塊差不多，讀 4 byte ，滿足條件再繼續 Decrypt 下個區塊

繼續推出 license.drm 的 5~8 位為 `0xE7, 0xEE, 0x66, 0x52`
用一樣的方式 dump 下一個區塊:

```
0x00000176: VM_fgetc
0x00000177: VM_Encode [SP]
0x00000178: NOT [SP]
0x00000179: XOR [SP], 0xfe
0x0000017B: VM_fgetc
0x0000017C: XOR [SP], 0xe1
0x0000017E: VM_Encode [SP]
0x0000017F: XCHG [SP], [SP - 01]
0x00000181: VM_fgetc
0x00000182: XOR [SP], 0xde
0x00000184: VM_Encode [SP]
0x00000185: VM_fgetc
0x00000186: XOR [SP], 0xad
0x00000188: VM_Encode [SP]
0x00000189: NOT [SP]
0x0000018A: XCHG [SP], [SP - 02]
0x0000018C: XCHG [SP], [SP - 01]
0x0000018E: VM_Encode [SP]
0x0000018F: XCHG [SP], [SP - 02]
0x00000191: CMP [SP], 0x91
0x00000193: JNE 0x1a0
0x00000195: CMP [SP], 0x5d
0x00000197: JNE 0x1a1
0x00000199: CMP [SP], 0x61
0x0000019B: JNE 0x1a1
0x0000019D: CMP [SP], 0xe6
0x0000019F: JE 0x1b4
0x000001A1: PUSH 0xa
0x000001A3: PUSH 0x21 ( ! )
0x000001A5: PUSH 0x65 ( e )
0x000001A7: PUSH 0x70 ( p )
0x000001A9: PUSH 0x6f ( o )
0x000001AB: PUSH 0x4e ( N )
0x000001AD: VM_fputc
(略)
0x0000022C: VM_Decrypt len: 337 addr: 0x231
0x00000231: Unknown op: 0x8a
0x00000232: Unknown op: 0x98
0x00000233: Unknown op: 0xf
(下略)
```

這邊執行了 VM_Encode 指令，稍微翻譯一下

```
VM_Encode:
n = (vm_stack[vm_sp] & 1) << 7;
n |= (vm_stack[vm_sp] & 0x10) >> 1;
n |= 2 * (vm_stack[vm_sp] & 8);
n |= (vm_stack[vm_sp] & 0x40) >> 5;
n |= 8 * (vm_stack[vm_sp] & 4);
n |= (vm_stack[vm_sp] & 0x80) >> 7;
n |= 32 * (vm_stack[vm_sp] & 2);
n |= (vm_stack[vm_sp] & 0x20) >> 3;
vm_stack[vm_sp] = n;
```

懶得寫 Decode ，所以直接暴力窮舉出 license.drm 的 9~12 位:

```py
#!/usr/bin/env python
# encoding: utf-8

def Encode(s):
     n = (s & 1) << 7
     n |= (s & 0x10) >> 1
     n |= 2 * (s & 8)
     n |= (s & 0x40) >> 5
     n |= 8 * (s & 4)
     n |= (s & 0x80) >> 7
     n |= 32 * (s & 2)
     n |= (s & 0x20) >> 3
     return n

for i in range(0xFF+1):
    if Encode(i ^ 0xE1) == 0xE6:
        print "key[9]:", hex(i)
    elif Encode(Encode(i ^ 0xDE)) == 0x61:
        print "key[10]:", hex(i)
    elif (~Encode(i) ^ 0xFE) & 0xFF == 0x5D:
        print "key[8]:", hex(i)
    elif ~Encode(i ^ 0xAD) & 0xFF  == 0x91:
        print "key[11]:", hex(i)
```

結果為 `0x3A, 0x86, 0xBF, 0xDB`
繼續 Decrypt 下一個區塊: 

```
0x00000231: VM_fgetc
0x00000232: XOR [SP], 0xde
0x00000234: XOR [SP], 0x20
0x00000236: XOR [SP], 0x8
0x00000238: VM_fgetc
0x00000239: XOR [SP], 0xad
0x0000023B: NSWAP [SP]
0x0000023C: NOT [SP]
0x0000023D: VM_fgetc
0x0000023E: VM_Encode [SP]
0x0000023F: XOR [SP], 0x4
0x00000241: XOR [SP], 0x2
0x00000243: XOR [SP], 0xbe
0x00000245: VM_fgetc
0x00000246: NXCHG [SP], [SP - 02]
0x00000248: XOR [SP], 0xef
0x0000024A: NOT [SP]
0x0000024B: XCHG [SP], [SP - 03]
0x0000024D: XCHG [SP], [SP - 02]
0x0000024F: XCHG [SP], [SP - 01]
0x00000251: CMP [SP], 0xb8
0x00000253: JNE 0x260
0x00000255: CMP [SP], 0xbb
0x00000257: JNE 0x261
0x00000259: CMP [SP], 0xc6
0x0000025B: JNE 0x261
0x0000025D: CMP [SP], 0xf6
0x0000025F: JE 0x274
0x00000261: PUSH 0xa
0x00000263: PUSH 0x21 ( ! )
0x00000265: PUSH 0x65 ( e )
0x00000267: PUSH 0x70 ( p )
0x00000269: PUSH 0x6f ( o )
0x0000026B: PUSH 0x4e ( N )
0x0000026D: VM_fputc
0x0000026E: VM_fputc
(略)
0x000002E6: VM_Decrypt len: 145 addr: 0x2eb
0x000002EB: Unknown op: 0x48
0x000002EC: Unknown op: 0xc9
0x000002ED: Unknown op: 0xff
(下略)
```

這塊比較麻煩的是 NXCHG 指令，互換 nibble 的部分
一樣可以用窮舉快速解決:

```py
#!/usr/bin/env python
# encoding: utf-8

def Encode(s):
     n = (s & 1) << 7;
     n |= (s & 0x10) >> 1;
     n |= 2 * (s & 8);
     n |= (s & 0x40) >> 5;
     n |= 8 * (s & 4);
     n |= (s & 0x80) >> 7;
     n |= 32 * (s & 2);
     n |= (s & 0x20) >> 3;
     return n
 
def NXCHG(x, y):
    return x & 0xF0 | y & 0x0F
 
def NSWAP(x):
    return x >> 4 | (x << 4 & 0xFF)
 
for i in range(0xFF+1):
    if i ^ 0xde ^ 0x20 ^ 0x08 == 0xc6:
        print "key[12]:", hex(i)
    elif Encode(i) ^ 0x04 ^ 0x02 ^ 0xbe == 0xb8:
        print "key[14]:", hex(i)
 
for i in range(0xFF+1):
    for j in range(0xFF+1):
        _i = (~(NSWAP(i ^ 0xad)) & 0xFF)
        if NXCHG(_i, j) == 0xbb and ~(NXCHG(j, _i) ^ 0xef) & 0xFF == 0xf6:
            print "key[13]", hex(i)
            print "key[15]", hex(j)
```

得到最後 4 位 Key 為 `0x30, 0x39, 0x00, 0xEB`
完整的 Key 為  `0x5F, 0x9C, 0x47, 0x01, 0xE7, 0xEE, 0x66, 0x52, 0x3A, 0x86, 0xBF, 0xDB, 0x30, 0x39, 0x00, 0xEB`

寫進 license.drm 再執行程式:

```sh
$ python -c "open('license.drm', 'wb').write('\x5F\x27\x8B\xDC\xE7\xEE\x66\x52\x3A\x86\xBF\xDB\x30\x39\x00\xEB')"
$ ./vm-2fbed3f5a894d56be6b2ba328f9e2411
Stage 1 complete! That was easy, wasn&#39;t it?
Stage 2 complete! Keep moving!
Stage 3 complete! You are nearly there!
Stage 4 complete! Hope you liked it.

Now you can haz key: 'Vm_ReVeRsInG_Is_FuN'
```

FLAG: `Vm_ReVeRsInG_Is_FuN`
