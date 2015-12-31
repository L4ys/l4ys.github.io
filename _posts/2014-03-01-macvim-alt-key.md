---
layout: post
title: 在 MacVim 設置 alt key 快捷鍵
category: others
---

前陣子在改 mac 上的 vimrc 時，發現 alt key (option) 在 OSX 上沒辦法被映射
像是下面這個用到 alt+j / alt+k 的快捷鍵就無法使用

```vim
" Move a line of text using ALT+[jk]
nmap <M-j>; mz:m+<cr>`z
nmap <M-k> mz:m-2<cr>`z
vmap <M-j> :m'>+<cr>`<my`>mzgv`yo`z
vmap <M-k> :m'<-2<cr>`>my`<mzgv`yo`z
```

後來發現了一個神奇的方式可以在 OSX 上成功設置 alt key 組合鍵

<!--more-->

在 OSX 按住 alt 再輸入其他文字，會出現特殊符號，像是 `åß∂ƒ©˙∆˚¬Ω≈ç` 等等

而 `option + j = ∆` / `option + k = ˚`

因此我們可以寫以下設定來達成 alt 組合鍵:

```vim
" Move a line of text using ALT+[jk]
nmap <M-j> mz:m+<cr>`z
nmap <M-k> mz:m-2<cr>`z
vmap <M-j> :m'>+<cr>`<my`>mzgv`yo`z
vmap <M-k> :m'<-2<cr>`>my`<mzgv`yo`z

if has("mac")
  " a small hack on macvim
  nmap ∆ <M-j>
  nmap ˚ <M-k>
  vmap ∆ <M-j>
  vmap ˚ <M-k>
endif
```

然後就能使用 alt + k / alt + j 來移動整行文字了，其他組合鍵也可以使用相同的方式設定

