---
layout: post
title: Trend Micro CTF 2015 Final - Binary 2
category: writeup
---

有點煩人的一題，在二天時出現，比賽時在 Binary1 時間截止後才開始看這題，
可惜最後還是沒能在時間內完成

<!--more-->

[Download Link](https://www.dropbox.com/s/w5j6vx8fmorddxc/TM_B2.rar?dl=0)

題目給了 `MISC.exe` 跟 `voice.bin`，exe 是 32bit PE ，用 `VMProtect` 加過殼

*順帶一提，TMCTF Final 的 Binary 題目是透過隨身碟提供的，而提交 Flag 則需要用筆寫在答案紙上交到後方裁判台，相當有趣...*

---

## Stage 1
MISC.exe 執行後會呼叫 Windows Speech API，開始以每次兩個字的方式唸出一串英文數字，
仔細聽前幾個 byte 會發現是 `7F454C46`，所以應該就是在唸一個 ELF 的 hex dump

![MISC.exe](http://i.imgur.com/Q8XDIRg.png)

接下來的問題就是該如何 Dump 出 ELF，在分析過 `MISC.exe` 之後，
發現沒辦法簡單在記憶體中找到呼叫 Speech API 時的參數字串，同時呼叫 Speech API 的關鍵部分也被殼 VM 掉了，
逆向起來相當不容易

最後決定將目標轉移到 Speech API 身上，Hook sapi.dll 中的 Speak() 並 Dump 出參數字串

由於 sapi.dll 並沒有 export Speak()，這裡可以自己寫支程式呼叫 Speak() 來快速定位出 Speak() 的位置

SAPI 的呼叫可參考 [MSDN](https://msdn.microsoft.com/en-us/library/ms720163%28v=vs.85%29.aspx)

接下來寫個 DLL 做 Memory Patch 來 Dump ELF:

```cpp
#include <Windows.h>
#include <stdio.h>
#include <time.h>

#define LOG_FILE ".\\log.txt"

void WriteLog(LPCWSTR format, ...)
{
    va_list  valist;
    FILE *fp;

    if (fopen_s(&fp, LOG_FILE, "a"))
        return;

    va_start(valist, format);
    vfwprintf(fp, format, valist);
    va_end(valist);
    fclose(fp);
}

void SetCall(DWORD dwSource, LPVOID lpTarget, UINT uNops = 0)
{
    DWORD OldProtect, OldProtect2;
    VirtualProtect((LPVOID)dwSource, 5 + uNops, PAGE_READWRITE, &OldProtect);
    *(LPBYTE)dwSource = 0xE9;
    *(LPDWORD)(dwSource + 1) = (DWORD)lpTarget - dwSource - 5;
    memset((LPVOID)(dwSource + 5), 0x90, uNops);
    VirtualProtect((LPVOID)dwSource, 5 + uNops, OldProtect, &OldProtect2);
}

wchar_t* p;
__declspec(naked) void patch()
{
    __asm {
        MOV EAX, DWORD PTR DS:[ESP+8]
        MOV p, EAX
        PUSHAD
    }
    WriteLog(L"%s", p);
    __asm {
        POPAD
        RET 0x10
    }
}

bool WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    switch (fdwReason)
    {
    case DLL_PROCESS_ATTACH:
    {
        DisableThreadLibraryCalls(hinstDLL);
        DeleteFileA(LOG_FILE);
        WriteLog(L"DLL Loaded!\n");

        HMODULE hSAPI = LoadLibraryA("C:\\Windows\\System32\\Speech\\Common\\sapi.dll");
        SetCall((DWORD)hSAPI + 0x0a7fbe, patch, 0);

        break;
    }
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
        break;
    case DLL_PROCESS_DETACH:
        WriteLog(L"\nDLL Unloaded.\n");
        break;
    }
    return true;
}

```

配合 OllyDBG + StrongOD 進行 DLL Injection，最後將 Dump 出的字串轉存為 ELF 檔案

---

## Stage 2

```bash
elf.out: ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.26, BuildID[sha1]=1a39aceebdd222960baa559aaa720c7be2645bae, stripped
```

ELF 非常小，丟進 IDA Pro 分析:

```c
int __cdecl main()
{
  int i; // eax@1
  signed int v1; // ecx@3
  int v2; // esi@3
  int v3; // ebx@5
  signed int v4; // eax@6
  unsigned __int8 v5; // di@6
  int v6; // edx@7
  char v7; // cl@7
  unsigned __int8 v9; // [sp+1Eh] [bp-11Eh]@6
  char v10; // [sp+1Fh] [bp-11Dh]@6
  signed int v11; // [sp+23h] [bp-119h]@1
  signed int v12; // [sp+27h] [bp-115h]@1
  char v13; // [sp+2Bh] [bp-111h]@1
  char v14[256]; // [sp+2Ch] [bp-110h]@2
  char v15; // [sp+12Ch] [bp-10h]@3
  unsigned __int8 v16; // [sp+12Dh] [bp-Fh]@3

  i = 0;
  v11 = "ssap";
  v12 = "drow";
  v13 = 0;
  do
  {
    v14[i] = i;
    ++i;
  }
  while ( i != 256 );
  v1 = 'p';
  v15 = 0;
  LOWORD(i) = 0;
  v2 = 0;
  v16 = 0;
  while ( 1 )
  {
    v3 = v14[i];
    v2 += v1 + v3;
    v14[i++] = v14[v2];
    v14[v2] = v3;
    if ( i == 256 )
      break;
    v1 = *(&v11 + (i & 7));
  }
  v4 = 1;
  v5 = 0;
  v10 = v15;
  v9 = v16;
  do
  {
    v6 = (v4 + v10);
    v7 = v14[v6];
    v9 += v7;
    v14[v6] = v14[v9];
    v14[v9] = v7;
    LOBYTE(v6) = byte_804913F[v4++] ^ v14[(v14[v6] + v7)];
    v5 ^= v6;
  }
  while ( v4 != 190548 );
  putchar(v5);
  return 0;
}
```

分析後發現程式本身沒有讀取任何資料，看起來在做某種解密之類的運算，
在 `main()` 尾端的 v5 ^= v6; (0x8048414) 處下斷點，會發現迴圈中前四次 EDX 的值會是 'R', 'a', 'r', '!'

直接寫個 pintool 來快速 Dump 執行到 0x8048414 時的 EDX:

```cpp
#include "pin.H"
#include <fstream>
#include <iomanip>
#include <iostream>

ofstream OutFile;
KNOB<string> KnobOutputFile(KNOB_MODE_WRITEONCE, "pintool", "o", "dump2.txt", "specify output file name");

VOID DumpEDX(ADDRINT EDX)
{
    OutFile << hex << setfill('0') << setw(2) << EDX;
}

VOID insert_hooks(INS ins, VOID *val)
{
    if ( INS_Address(ins) == 0x8048414 ) {
        INS_InsertCall(
            ins, IPOINT_BEFORE, (AFUNPTR)DumpEDX,
            IARG_REG_VALUE, REG_EDX,
            IARG_END);
    }
}

VOID Fini(INT32 code, VOID *v)
{
    OutFile.close();
}

int main(int argc, char *argv[])
{
    PIN_Init(argc, argv)

    OutFile.open(KnobOutputFile.Value().c_str());

    INS_AddInstrumentFunction(insert_hooks, NULL);
    PIN_AddFiniFunction(Fini, 0);
    PIN_StartProgram();

    return 0;
}
```

最後會得到一個有密碼的 rar 壓縮檔

## Stage 3
經過暴力破解或猜測後會發現密碼是 `password`，解壓縮得到一張 `1_stego.png`
![](http://i.imgur.com/HgLX1pY.png)

## Stage 4
仔細觀察 rgba 值會發現第三個 bit 似乎有藏資訊，寫個 script 以 column 順序可以拉出另一張 png:

![](http://i.imgur.com/SgVV3lU.png)

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from PIL import Image

im = Image.open("./1_stego.png")
pix = im.load()
(w, h) = im.size
out = []
for x in range(w):
    for y in range(h):
        r,g,b,a = pix[x,y]

        if a != 255:
            out.append(str(int(a & 8 == 8)))
            out.append(str(int(r & 8 == 8)))
            out.append(str(int(g & 8 == 8)))
            out.append(str(int(b & 8 == 8)))

o = ""
for i in range(0, len(out), 8):
    o += chr(int("".join(out[i:i+8]), 2))
open("out.png", "wb").write(o)

```

## Stage 5
第二張 png 觀察後一樣發現 LSB 看起來藏了東西，用一樣的方式 Dump 出來，
最後得到一個帶密碼的 zip，flag.txt 就在裡面

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from PIL import Image

im = Image.open("./out.png")
pix = im.load()
(w, h) = im.size
out = []
for x in range(w):
    for y in range(h):
        r,g,b,a = pix[x,y]
        out.append(str(a & 1))
        out.append(str(r & 1))
        out.append(str(g & 1))
        out.append(str(b & 1))

o = ""
for i in range(0, len(out), 8):
    o += chr(int("".join(out[i:i+8]), 2))
open("out.zip", "wb").write(o)

```

## Stage 6
Hex Editor 看一下 zip 會發現有個字串 `tmctf2015` ，拿來當密碼就可以解壓縮出 flag.txt 了

Flag: `tmctf{have_fun_with_tmctf_^o^}`

---

雖然題目神煩，包含了 Windows / Linux / Stego 

但 217 還是在時間內解出來了，只能說自己實作的速度實在太慢了 Q__Q
