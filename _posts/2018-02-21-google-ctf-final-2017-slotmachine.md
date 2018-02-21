---
layout: post
title: Google CTF Final 2017 - Slot Machine
category: writeup
tags: [reverse]
---

這次到瑞士參加 Google CTF 決賽，主辦方很用心的設計了一道硬體 Reverse 題
雖然解題過程中因為缺乏相關知識而踩了不少雷，但還是蠻有趣的

<!--more-->

### 0x00. Slot Machine

第一天比賽進行到一半，主辦方給了每個隊伍一台吃角子老虎機
看起來只是在現成的外殼內裝了一個開發版，可以透過 USB 供電啟動
> 賽後主辦單位表示他們盡千辛萬苦才收集到十多個古老的老虎機外殼給每個隊伍當題目 XD

![](https://i.imgur.com/emOvJRi.png)
![](https://i.imgur.com/0oEWRSm.png)

附贈麵包板跟一個 `Arduino ISP`，還貼心地附上一支螺絲起子...
> ISP 為 In-System Programmer，就是所謂的燒錄器，用來從開發版燒錄及讀取韌體


![](https://i.imgur.com/JE5jAXQ.png)


對應的是兩題 Reverse:

![](https://i.imgur.com/CLEfzM4.png)

![](https://i.imgur.com/5HUFIra.png)



以往打 CTF 看到硬體題幾乎都是無視

但晚上回到飯店後，在 atdog 的慫恿下覺得好像很有趣就一起研究了起來...

---

### 0x01. Hardware

用螺絲起子拆開外殼後，從 IC 上印的文字可以看出是 Atmel 的 `ATtiny88-PU`:
下方的排線則連接著前面板的按鈕

![](https://i.imgur.com/JL1xXQq.png)

> 帶回飯店拆開才發現裡面還塞了一塊磚頭...


電路板可能是自己設計的，拔下 IC 後發現板子上面還幽默的印了 `No flag here`
![](https://i.imgur.com/Gw7Vyuz.png)


---

### 0x02. Firmware

接著花了許多時間研究如何從 IC 中 dump 出 ROM

首先用麵包板將 ISP 上的 `MOSI` `VCC` `GND` `RESET` `SCK` `MISO` 接到 IC 對應的腳位上

![](https://i.imgur.com/k3yVzU3.png)


![](https://i.imgur.com/OTCb4wf.jpg)

![](https://i.imgur.com/Mt5YhX5.png)


接好線後，透過 `avrdude` 就可以 dump 出 ROM:

```bash
$ avrdude -p t88 -P usb -c usbasp -U flash:r:flash.bin:r

avrdude: warning: cannot set sck period. please check for usbasp firmware update.
avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.00s

avrdude: Device signature = 0x1e9311 (probably t88)
avrdude: reading flash memory:

Reading | ################################################## | 100% 4.22s

avrdude: writing output file "flash.bin"

avrdude: safemode: Fuses OK (E:FF, H:DF, L:EE)

avrdude done.  Thank you.
```

---

### 0x03. IDA Pro

拿到了 ROM，想用 IDA Pro 逆向時發現 IDA 沒有支援 `ATtiny88`

於是參考 [datasheet](http://ww1.microchip.com/downloads/en/DeviceDoc/doc8008.pdf) 自己擴充了 `avr.cfg`

照著 datasheet 定義 RAM / ROM 大小 / Interrupt Vector Table 以及 IO Port
```
.ATtiny88
SUBARCH=25
;doc8008.pdf
;

RAM=512
ROM=8192
EEPROM=64

; MEMORY MAP
; From Table 5-2. Layout of Data Memory and Register Area.
area DATA GPWR_        0x0000:0x0020   General Purpose Working Registers
area DATA FSR_         0x0020:0x0060   I/O registers
area DATA EXTIO_       0x0060:0x0100   I/O registers
area DATA I_SRAM       0x0100:0x0300   Internal SRAM

; Interrupt and reset vector assignments
; From Table 9-1. Reset and Interrupt Vectors in ATtiny48/88
entry __RESET        0x0000   External Pin, Power-on Reset, Brown-out Reset, Watchdog Reset, and JTAG AVR Reset
entry INT0_          0x0001   External Interrupt Request 0
entry INT1_          0x0002   External Interrupt Request 1
entry PCINT0_        0x0003   Pin Change Interrupt Request 0
entry PCINT1_        0x0004   Pin Change Interrupt Request 1
entry PCINT2_        0x0005   Pin Change Interrupt Request 2
entry PCINT3_        0x0006   Pin Change Interrupt Request 3
entry WDT            0x0007   Watchdog Time-out Interrupt
entry TIMER1_CAPT    0x0008   Timer/Counter1 Capture Event
entry TIMER1_COMPA   0x0009   Timer/Counter1 Compare Match A
entry TIMER1_COMPB   0x000A   Timer/Counter1 Compare Match A
entry TIMER1_OVF     0x000B   Timer/Counter1 Overflow
entry TIMER0_COMPA   0x000C   Timer/Counter0 Compare Match A
entry TIMER0_COMPB   0x000D   Timer/Counter0 Compare Match B
entry TIMER0_OVF     0x000E   Timer/Counter0 Overflow
entry SPI_STC        0x000F   Serial Transfer Complete
entry ADC            0x0010   ADC Conversion Complete
entry EE_READY       0x0011   EEPROM Ready
entry ANALOG_COMP    0x0012   Analog Comparator
entry TWI            0x0013   2-wire Serial Interface

; INPUT/OUTPUT PORTS
; From 24. Register Summary
RESERVED0000    0x0000
RESERVED0001    0x0001
RESERVED0002    0x0002
PINB            0x0003
PINB.PINB7       7
PINB.PINB6       6
PINB.PINB5       5
PINB.PINB4       4
PINB.PINB3       3
PINB.PINB2       2
PINB.PINB1       1
PINB.PINB0       0
DDRB            0x0004
DDRB.DDB7        7
DDRB.DDB6        6
DDRB.DDB5        5
DDRB.DDB4        4
DDRB.DDB3        3
DDRB.DDB2        2
DDRB.DDB1        1
DDRB.DDB0        0
PORTB           0x0005
PORTB.PORTB7     7
PORTB.PORTB6     6
PORTB.PORTB5     5
PORTB.PORTB4     4
PORTB.PORTB3     3
PORTB.PORTB2     2
PORTB.PORTB1     1
PORTB.PORTB0     0
PINC            0x0006
PINC.PINC7       7
PINC.PINC6       6
PINC.PINC5       5
PINC.PINC4       4
PINC.PINC3       3
PINC.PINC2       2
PINC.PINC1       1
PINC.PINC0       0
DDRC            0x0007
DDRC.DDC7        7
DDRC.DDC6        6
DDRC.DDC5        5
DDRC.DDC4        4
DDRC.DDC3        3
DDRC.DDC2        2
DDRC.DDC1        1
DDRC.DDC0        0
PORTC           0x0008
PORTC.PORTC7     7
PORTC.PORTC6     6
PORTC.PORTC5     5
PORTC.PORTC4     4
PORTC.PORTC3     3
PORTC.PORTC2     2
PORTC.PORTC1     1
PORTC.PORTC0     0
PIND            0x0009
PIND.PIND7       7
PIND.PIND6       6
PIND.PIND5       5
PIND.PIND4       4
PIND.PIND3       3
PIND.PIND2       2
PIND.PIND1       1
PIND.PIND0       0
DDRD            0x000A
DDRD.DDD7        7
DDRD.DDD6        6
DDRD.DDD5        5
DDRD.DDD4        4
DDRD.DDD3        3
DDRD.DDD2        2
DDRD.DDD1        1
DDRD.DDD0        0
PORTD           0x000B
PORTD.PORTD7     7
PORTD.PORTD6     6
PORTD.PORTD5     5
PORTD.PORTD4     4
PORTD.PORTD3     3
PORTD.PORTD2     2
PORTD.PORTD1     1
PORTD.PORTD0     0
PINA            0x000C
PINA.PINA3       3
PINA.PINA2       2
PINA.PINA1       1
PINA.PINA0       0
DDRA            0x000D
DDRA.DDA3        3
DDRA.DDA2        2
DDRA.DDA1        1
DDRA.DDA0        0
PORTA           0x000E
PORTA.PORTA3     3
PORTA.PORTA2     2
PORTA.PORTA1     1
PORTA.PORTA0     0
RESERVED000F    0x000F
RESERVED0010    0x0010
RESERVED0011    0x0011
PORTCR          0x0012
PORTCR.BBMD      7
PORTCR.BBMC      6
PORTCR.BBMB      5
PORTCR.BBMA      4
PORTCR.PUDD      3
PORTCR.PUDC      2
PORTCR.PUDB      1
PORTCR.PUDA      0
RESERVED0013    0x0013
RESERVED0014    0x0014
TIFR0           0x0015
TIFR0.OCF0B      2
TIFR0.OCF0A      1
TIFR0.TOV0       0
TIFR1           0x0016
TIFR1.ICF1       5
TIFR1.OCF1B      2
TIFR1.OCF1A      1
TIFR1.TOV1       0
RESERVED0017    0x0017
RESERVED0018    0x0018
RESERVED0019    0x0019
RESERVED001A    0x001A
PCIFR           0x001B
PCIFR.PCIF3      3
PCIFR.PCIF2      2
PCIFR.PCIF1      1
PCIFR.PCIF0      0
EIFR            0x001C
EIFR.INTF1       1
EIFR.INTF0       0
EIMSK           0x001D
EIMSK.INT1       1
EIMSK.INT0       0
GPIOR0          0x001E
EECR            0x001F
EECR.EEPM1       5
EECR.EEPM0       4
EECR.EERIE       3
EECR.EEMPE       2
EECR.EEPE        1
EECR.EERE        0
EEDR            0x0020
EEARL           0x0021
RESERVED0022    0x0022
GTCCR           0x0023
GTCCR.TSM        7
GTCCR.PSRSYNC    0
RESERVED0024    0x0024
TCCR0A          0x0025
TCCR0A.CTC0      3
TCCR0A.CS02      2
TCCR0A.CS01      1
TCCR0A.CS00      0
TCNT0           0x0026
OCR0A           0x0027
OCR0B           0x0028
RESERVED0029    0x0029
GPIOR1          0x002A
GPIOR2          0x002B
SPCR            0x002C
SPCR.SPIE        7
SPCR.SPE         6
SPCR.DORD        5
SPCR.MSTR        4
SPCR.CPOL        3
SPCR.CPHA        2
SPCR.SPR1        1
SPCR.SPR0        0
SPSR            0x002D
SPSR.SPIF        7
SPSR.WCOL        6
SPSR.SPI2X       0
SPDR            0x002E
RESERVED002F    0x002F
ACSR            0x0030
ACSR.ACD         7
ACSR.ACBG        6
ACSR.ACO         5
ACSR.ACI         4
ACSR.ACIE        3
ACSR.ACIC        2
ACSR.ACIS1       1
ACSR.ACIS0       0
DWDR            0x0031
RESERVED0032    0x0032
SMCR            0x0033
SMCR.SM1         2
SMCR.SM0         1
SMCR.SE          0
MCUSR           0x0034
MCUSR.WDRF       3
MCUSR.BORF       2
MCUSR.EXTRF      1
MCUSR.PORF       0
MCUCR           0x0035
MCUCR.BODS       6
MCUCR.BODSE      5
MCUCR.PUD        4
RESERVED0036    0x0036
SPMCSR          0x0037
SPMCSR.RWWSB     6
SPMCSR.CTPB      4
SPMCSR.RFLB      3
SPMCSR.PGWRT     2
SPMCSR.PGERS     1
SPMCSR.SELFPRGEN 0
RESERVED0038    0x0038
RESERVED0039    0x0039
RESERVED003A    0x003A
RESERVED003B    0x003B
RESERVED003C    0x003C
SPL             0x003D
SPL.SP7          7
SPL.SP6          6
SPL.SP5          5
SPL.SP4          4
SPL.SP3          3
SPL.SP2          2
SPL.SP1          1
SPL.SP0          0
SPH             0x003E
SPH.SP9          1
SPH.SP8          0
SREG            0x003F
SREG.I           7   Global Interrupt Enable
SREG.T           6   Bit Copy Storage
SREG.H           5   Half Carry Flag
SREG.S           4   Sign Bit
SREG.V           3   Two's Complement Overflow Flag
SREG.N           2   Negative Flag
SREG.Z           1   Zero Flag
SREG.C           0   CarryFlag
```

接著就可以在 IDA Pro 選擇 ATtiny88:
![](https://i.imgur.com/sOzH85h.png)

---

### 0x04 Fix RAM

將 ROM load 進 IDA 後，還需要修正 RAM Segment 中的內容
由於全域變數等初始資料會在程式初始化時從 ROM 中被複製到 RAM 中
所以我們需要先從 ROM 中讀出這一塊資料並寫入到 IDA 的 RAM Segment 中

從 `__RESET` 中能夠看出 ROM 的哪部分會被搬進 RAM:
> 對照 avr-gcc 的 source code 會發現其實就是 `__do_clear_bss`

![](https://i.imgur.com/XbREerK.png)


> avr 的 R26:R27, R28:R29 及 R30:R31 這些 Register 有著額外的意義
> 分別對應到 X, Y, Z 這三個 16-bit Pointer Register
> avr 的每個 register 只有 1byte ，因此必須透過 X, Y, Z 來存取 SRAM 的位置


所以從 ROM 的 offset 0x19A6 開始的 0xDE bytes 即為 RAM 的內容
透過 IDAPython 把 RAM 的內容抓出來，並寫到 RAM Segment 的對應位置上
```python
with open("./flash.bin") as f:
    f.seek(0x19a6)
    ram = f.read(0xde)

for i, b in enumerate(ram):
    PatchByte(0x100100 + i, ord(b))
```

完成後從字串就可以看到一些跟 Flag 有關的內容:
![](https://i.imgur.com/nHk7pGM.png)

> 主辦方有說明給各隊伍的硬體中只會有測試 Flag
> 必須要破解裁判台上那一台吃角子老虎機才能得到真正的 Flag

但由於程式存取 RAM 的資料時會透過 X Y Z，所以還是沒辦法使用 cross reference 來直接找到程式引用字串的地方

---

### 0x05. Flag1

接下來就是慢慢的看著 avr assembly 分析程式...

> 由於程式的基本邏輯都在同一個龐大的函數中，分析起來相當困難，於是直接拿了現場的大螢幕來看 Graph，沒想到用起來居然意外的順手:
![](https://i.imgur.com/vHsJ9TK.jpg)



首先找到存取第一把 Flag 字串的位置，慢慢的往回追
![](https://i.imgur.com/NpShqJl.png)

最後發現程式會記錄下九次按鈕的輸入，並與特定值比對:
![](https://i.imgur.com/l6Y5al8.png)

![](https://i.imgur.com/UhtbgIj.png)

因此只要照著 `1,2,1,2,5,4,3,4,3` 的順序依序按下按扭，就可以得到第一題的 Flag:

![](https://i.imgur.com/ZDhinbt.png)


Flag: `CTF{5UP3R_53CR37_BTN_C0MB1N4710N}`

---

### 0x06. Flag2

除了顯示 Flag1 的密碼外，輸入另一組密碼則可以進入開發者模式
允許我們輸入一個 7 位數的 Delta:

![](https://i.imgur.com/jE0wUl5.png)

但嘗試了很多次還是感受不到這個 Delta 能夠對遊戲造成什麼影響
反覆檢查了輸入的部分也不存在 Out-of-bound 之類的問題
只好繼續逆向

經過了一段時間的分析之後，發現有個地方使用了一個常出現於 [PRNG](https://en.wikipedia.org/wiki/Pseudorandom_number_generator) 的常數 `0x41C64E6D`，八成就是個 rand()

![](https://i.imgur.com/hLZYHr0.png)

於是馬上就跟前面的 Delta 聯想在一塊，這是要我們反推出 PRNG Seed 並透過 delta 控制遊戲結果!

---

### 0x07 PRNG

繼續往回追之後發現了決定盤面的方式:
![](https://i.imgur.com/yZELlny.png)


按下按鈕後，遊戲會透過 rand() 得到的結果所在範圍決定每一個格子的圖樣，程式邏輯如下:

```c
uint32_t seed = 0;

uint16_t rand()
{
    seed = 0x41c64e6d * seed + 31337;
    return ((seed >> 16) ^ (seed & 0xffff));
}

uint32_t get_rand_index()
{
    uint32_t r = rand();
    if (r >= 0xFD70)
        return 3;  // SEVEN
    else if (r >= 0xF0A3)
        return 2;  // LEMON
    else if (r >= 0xBAE0)
        return 1;  // BAR
    return 0;
}
```

知道了盤面決定方式之後我們可以建出所有 seed 會產生的初始盤面

玩幾次遊戲之後，從我們建立的表中找到對應的盤面順序，得到初始 seed
接著推算出要中 777 大獎所需的初始 seed 值:

![](https://i.imgur.com/YZorU2d.png)



最後再透過 Developer Menu 將加上兩個 seed 之間的 delta，就能夠中大獎了:


<video preload controls src="http://l4ys.tw/jackpot.mov"></video>


Solution: https://gist.github.com/L4ys/7fa83d74adf0f76030838764375f68b7


