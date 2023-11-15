+++
title = "按條件加載 Git 配置"
author = ["Hilton Chain"]
description = "git-config(1): Conditional includes."
date = 2023-08-25T01:49:00+08:00
lastmod = 2023-11-15T14:41:00+08:00
tags = ["Git"]
categories = ["notes"]
draft = false
image = "cover.jpg"
+++

## 緣起 {#緣起}

我的 Git 設置有要求對 commit 簽名，然而在 OpenPGP 智能卡方面卻又有每次簽名必須單獨驗證的設置。這對改動不多，在本地就能保證線性歷史的倉庫來說還好，但是對需要頻繁 cherry-pick + rebase 的就難說了。

雖然可以在倉庫內關掉簽名要求，但發佈時還是得保證簽名，爲此臨時手動開關選項稍有些麻煩，而我也無法說服自己調整智能卡設置就是了。

所以得找一個折中方案，能直接想到的是：

1.  仍然默認要求簽名。
2.  針對特定倉庫關閉簽名要求，並在其中設置一個要求簽名的分支（就叫 outgoing 吧）。

不過我不清楚第二點該如何完成，所幸搜索到了 Git 設置中 includeIf 的例子。


## RTFM {#rtfm}

在 Git 中有兩種從其他來源加載配置文件的方法，其一是 include，包含一個 path 選項，值爲以當前配置文件爲參照的相對路徑，形如：

```cfg
[include]
        path = ../etc/git/gitconfig
```

在 .git/config 中加入上述配置，就像是把這段配置替換爲了 etc/git/gitconfig（.git/../etc/git/gitconfig）中的內容。

另一種方法就是 includeIf 了，同前者一樣包含 path 選項，只不過除此以外還需要一個條件，只有滿足條件後纔會加載 path 中指定的配置文件。條件有很多種[^fn:1]，而我想要指定「切出要求簽名的分支（比如前面提到的 outgoing）時」，所以用到了 onbranch。寫出來像是這樣：

```cfg
[includeIf "onbranch:outgoing"]
        path = ../etc/git/gitconfig
```


## 結果 {#結果}

因此在需要設置的倉庫中如下操作即可：

```shell
# 關閉簽名要求
git config commit.gpgsign false

# 切出 outgoing 分支時讀取配置文件 outgoing
git config includeIf.onbranch:outgoing.path outgoing

# 在配置文件 .git/outgoing 中要求簽名
git config -f .git/outgoing commit.gpgsign true
```

生成的 Git 配置文件：

```cfg
# .git/config，有省略：
[commit]
        gpgsign = false
[includeIf "onbranch:outgoing"]
        path = outgoing
# .git/config 在此結束。
```

```cfg
# .git/outgoing：
[commit]
        gpgsign = true
# .git/outgoing 在此結束。
```

> 題圖作者：Pat Whelen（[Unsplash](https://unsplash.com/photos/brown-wood-log-in-close-up-photography-JIxBp37Vt3Y)）

[^fn:1]: 詳細參見 `man 1 git-config` 或 `info "(gitman) git-config"` 中 Conditional includes 部分。
