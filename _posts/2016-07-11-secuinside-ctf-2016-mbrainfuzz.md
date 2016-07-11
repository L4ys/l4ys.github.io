---
layout: post
title: SECUINSIDE CTF 2016 - mbrainfuzz / mbrainfuzz_returns
category: writeup
tags: [pwn]
---

不得不說這次比賽主辦方出了不少包

而 `mbrainfuzz` 跟 `mbrainfuzz_returns` 這兩題應該算是出包最多次的題目

應該不少隊伍對這兩題的怨念很深

從剛開賽就出現各種問題，主辦方前前後後大概修了七八次

最後似乎還是沒有完全修好...


`14:27 <@mango> mbrainfuzz has some problems, so shutting down..`
`15:01 <@setuid0> mbrainfuzz challenge is back`
`15:03 <@setuid0> forgot to change binary. sorry`
`17:09 < b2xiao> so we're on what, version 8 of mbrainfuzz at this point? I lost track...so many solver scripts, all useless...`

<!--more-->

---

### Challenge:

題目跟 [pwnable.kr](http://pwnable.kr/) 上的 `aeg` 有點類似

連上服務後會回傳一支隨機產生的 ELF ，從 `argv[1]` 讀取字串，

如果輸入的字串可以通過 80 個函數的檢查，就可以觸發一個 buffer overflow

因此要做的就是某種程度上的 AEG ，根據執行檔內容自動生成攻擊 payload

而兩個題目的差異在於 `mbrainfuzz` 沒有任何保護，`mbrainfuzz_returns` 是有 NX 的版本

---

題目在比賽中更新並簡化了很多次，最後的版本變得非常單純，每一個檢查函數長得都差不多:

```c
void __fastcall sub_4007FC(char a1, char a2, char a3, char a4)
{
  if ( a3 == 5 && a2 == 1 && a1 == 2 && a3 + 2 < a4 * abs(a3 + 2) )
    sub_4008A3(input[116], input[30], input[41], input[86]);
}

void __fastcall sub_4008A3(char a1, char a2, char a3, char a4)
{
  if ( a3 == 8 && a2 == 7 && a1 == 1 && a3 + 7 < a4 * abs(a3 - 3) )
    sub_40094C(input[58], input[53], input[148], input[66]);
}
...

```

通過所有的檢查就可以來到觸發 buffer overflow 的位置:

```c
void *sub_4041D4()
{
  char dest[16]; // [sp+10h] [bp-10h]@1

  return memcpy(dest, input[334], 367);
}
```

---

### Exploit:

直接從反組譯結果中找出每個檢查函數中的常數，可以很容易的產生出符合條件的 input

但題目所隨機產生的 ELF 約有 9 成是無解的，而就算送出正確的 payload ，

也有不小的機率會因為不知名原因無法觸發，需要跑個上百次才能成功拿到 shell...

#### mbrainfuzz:
[exploit for mbrainfuzz](https://gist.github.com/L4ys/73245e525433df23d599155ad7d3a806)

#### mbrainfuzz_returns:
有 NX 的版本，[ddaa](http://ddaa.tw/) 寫了偽造 `linkmap` 後透過 `ret-to-dl_resolve` 執行 `system` 的 exploit:
[exploit for mbrainfuzz_returns](https://gist.github.com/L4ys/660ad558fa0208e3d60d959394e0a6c6)

---

順便附上拿到 shell 之後取得的題目 source code:

- [source code of mbrainfuzz](https://gist.github.com/L4ys/5615ccaee3517ae9fafdd2209b77ca52)
- [source code of mbrainfuzz_returns](https://gist.github.com/L4ys/fc19a64dbf30324af78760cb9efed7e6)

---

個人覺得這兩題設計得沒有 `pwnable.kr` 的 `arg` 完整，

最初的版本中有 captcha 的字串檢查，但似乎有問題所以後來拿掉了，有點可惜




