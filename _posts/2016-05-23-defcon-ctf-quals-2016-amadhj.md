---
layout: post
title: DEFCON CTF QUALS 2016 - amadhj
category: writeup
tags: [reverse]
---


ELF64 reverse 題

程式行為很簡單，讀 32 byte ，當成四個 QWORD 個別進行一連串運算，

最後得到的 4 個值 xor 必須等於特定值，乍看之下就是個 z3 題

<!--more-->

---

但比賽時跟 lucas 將程式重新用 python implement 後

嘗試用 z3 求解卻遇到各種莫名其妙的問題...


賽後看到有隊伍使用 [KLEE](http://klee.github.io) 來解這題

> KLEE is a symbolic virtual machine built on top of the LLVM compiler infrastructure, and available under the UIUC open source license. 

因此寫了這篇 writeup 來記錄一下 用 KLEE 配合 hexrays 的快速解法!

--- 

### KLEE 環境準備:

最簡單的方法就是直接使用官方提供的的 [docker image](http://klee.github.io/docker/) 來建立環境:

```sh
# get image
$ docker pull klee/klee
```

```sh
# start container
$ docker run --rm -ti --ulimit='stack=-1:-1' klee/klee
klee@92c23029e8e7:~$ 
```

```sh
# check klee version
klee@92c23029e8e7:~$ klee --version
KLEE 1.2.0 (https://klee.github.io)
  Built May 18 2016 (11:29:06)
  Build mode: Release+Asserts
  Build revision: unknown

LLVM (http://llvm.org/):
  LLVM version 3.4

  Optimized build.
  Built Mar  5 2014 (17:05:10).
  Default target: x86_64-pc-linux-gnu
  Host CPU: corei7-avx

```

---

### Compile LLVM bitcode with clang

為了要產生 LLVM bitcode，得先將程式翻譯回 c code

先透過 IDA Pro 的 hexrays decompiler 產生關鍵部分的程式碼

> 小技巧: include IDA Pro plugin 目錄下的 defs.h 可省下許多修改 variable type 的時間 )

再加入 KLEE 的相關呼叫 ( 在原先印 Flag 處呼叫 klee_assert() 、 加入 symbolic variable 等等 ):

[https://gist.github.com/L4ys/7ee828a8570daabffb52583286780e5f](https://gist.github.com/L4ys/7ee828a8570daabffb52583286780e5f)

接著以 clang compile 得到 LLVM bitcode:

```sh
klee@92c23029e8e7:~$ clang -emit-llvm -c -g amadhj.c -I ~/klee_src/include/
```

### Magic

最後執行 klee ，等待 assert 被 trigger:

```sh
klee@a34c0d6f0dc7:~/amadhj$ klee ./amadhj.bc
KLEE: WARNING: cannot create klee-last symlink: Operation not supported
KLEE: output directory is "/home/klee/amadhj/./klee-out-0"
Using STP solver backend
KLEE: ERROR: /home/klee/amadhj/amadhj.c:549: ASSERTION FAIL: 0
KLEE: NOTE: now ignoring this error at this location

KLEE: done: total instructions = 36454
KLEE: done: completed paths = 2
KLEE: done: generated tests = 2
```

找到觸發 assert 的 testcase ，用 ktest-tool 列出資訊:

```sh
klee@a34c0d6f0dc7:~/amadhj$ ls ./klee-out-0/ | grep assert
test000002.assert.err

klee@a34c0d6f0dc7:~/amadhj$ ktest-tool ./klee-out-0/test000002.ktest
ktest file : './klee-out-0/test000002.ktest'
args       : ['./amadhj.bc']
num objects: 1
object    0: name: b'input'
object    0: size: 32
object    0: data: b'kj   kr  n ZC YV kykBn  Zdk Inxi'
```

將得到的 input 餵給題目，得到 Flag:

```sh
$ echo -n "kj   kr  n ZC YV kykBn  Zdk Inxi" | nc amadhj_b76a229964d83e06b7978d0237d4d2b0.quals.shallweplayaga.me 4567
The flag is: Da robats took err jerbs.
```

### Reference:

 - [http://pastebin.com/XG40a20H](http://pastebin.com/XG40a20H)

