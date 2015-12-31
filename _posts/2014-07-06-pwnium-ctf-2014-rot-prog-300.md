---
layout: post
title: Pwnium CTF 2014 - ROT
category: writeup
---

> Rot 90, Rot -90, Rot 90, Rot -90... nc 41.231.53.40 9090</blockquote>

題目名稱是 ROT，但跟 ROT 加解密完全沒關係，只是簡單的圖片操作題

<!--more-->

連上題目後會得到一大串編碼，然後問 Answer
每一次的編碼內容都不同，必須在約 3 至 4 秒內回傳答案:

![code](http://1.bp.blogspot.com/-txwx6QtB2xw/U7j0QTuoiXI/AAAAAAAAAXA/2dshELsPrO0/s1600/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7+2014-07-06+%E4%B8%8B%E5%8D%883.00.35.png)

base64 decode 後內容是一張 png

![png](http://4.bp.blogspot.com/-0B09NeozO1s/U7j0_ayLhNI/AAAAAAAAAXI/qiVmFH2oDUc/s1600/1.png)

可以看出其中有被切割過的文字，而題目名稱的 ROT 其實指的是 Rotate

( 在解這題的時候腦袋不清楚跑去切三角形...轉了半天，寫這篇的時候才發現只要垂直切一半再旋轉就好了... )

![](http://3.bp.blogspot.com/-Zg2PHP8WamE/U7j37J2a5CI/AAAAAAAAAXY/3HG_M8LDTXA/s1600/%25E6%259C%25AA%25E5%2591%25BD%25E5%2590%258D-1.png)

![](http://4.bp.blogspot.com/-gRzoCuA_IlQ/U7j49VKZSfI/AAAAAAAAAXk/xmJlY_lvI2A/s1600/2.png)

知道圖片怎麼拼後，剩下就是做 OCR ，得到圖中的文字，回傳給 Server
找了一些 library 後，最後決定用 pyocr 配合 tesseract 做 OCR
切圖及疊圖交給 PIL 處理

結果大概長這樣，辨識度挺不錯的
![result](http://1.bp.blogspot.com/-yyWo0AIyqxU/U7j6qpitVvI/AAAAAAAAAXs/uqaIVek3Eug/s1600/rot.png)

完整的 script:

```py
#!/usr/bin/python
import base64
import socket
import pyocr
import pyocr.builders
from PIL import Image

def removecolor(img, color):
    img = img.convert("RGBA")
    datas = img.getdata()

    newData = []
    for item in datas:
        if item[0] == color[0] and item[1] == color[1] and item[2] == color[2]:
            newData.append((255, 255, 255, 0))
        else:
            newData.append(item)

    img.putdata(newData)
    return img

# Prepare OCR
tool = pyocr.get_available_tools()[0]
lang = tool.get_available_languages()[0]

# Download png
print "Connecting..."
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("41.231.53.40", 9090))
d = s.recv(102400)
d += s.recv(102400)
d += s.recv(102400)
d += s.recv(102400)
d += s.recv(102400)
d = d[:d.find("Answer:")]

print "Saving..."

with f.open("1.png", "w"):
    f.write(base64.b64decode(d))

img = Image.open('1.png')

# Remove background
img = removecolor(img, (255, 255, 0))
img = removecolor(img, (0, 255, 255))
img = removecolor(img, (0, 255, 0))
img = removecolor(img, (255, 255, 255))
rst = Image.new("RGBA", (200,100), "white")

# Crop and rotate
left = img.crop((0, 0, 100, 200))
left = left.transpose(Image.ROTATE_90)
rst.paste(left, (0,0), left)

right = img.crop((100, 0, 200, 200))
right = right.transpose(Image.ROTATE_270)
rst.paste(right, (0,0), right)

# OCR
answer = tool.image_to_string(rst, lang=lang, builder=pyocr.builders.TextBuilder()).replace(" ","")
print "OCR Result: %s" % answer

s.send(answer + "\n")

# Recv flag
print s.recv(1024)

```

![flag](http://3.bp.blogspot.com/-ww473Uixvhg/U7j_ibOSPGI/AAAAAAAAAX8/xdgac5vNMVc/s1600/%E8%9E%A2%E5%B9%95%E5%BF%AB%E7%85%A7+2014-07-06+%E4%B8%8B%E5%8D%883.46.22.png)

