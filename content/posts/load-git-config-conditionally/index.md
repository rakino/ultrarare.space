+++
title = "按條件加載 Git 配置"
author = ["Hilton Chain"]
description = "git-config(1): Conditional includes."
date = 2023-08-25T01:49:00+08:00
lastmod = 2023-08-25T11:57:00+08:00
tags = ["Git"]
categories = ["notes"]
draft = false
+++

我的 Git 設置有要求對 commit 簽名，然而在 OpenPGP 智能卡方面卻又有每次簽名必須單獨驗證的設置。這對改動不多，在本地就能保證線性歷史的倉庫來說還好，但是對需要頻繁 cherry-pick + rebase 的就難說了。雖然可以在倉庫內關掉簽名要求，但發佈時還是得保證有簽名，爲此臨時開關選項稍有些麻煩，而我也無法說服自己調整智能卡設置就是了。

所以只能找一個折中方案，能直接想到的是：

1.  仍然默認要求簽名。
2.  在特定倉庫關閉簽名要求，並設置一個要求簽名的分支（e.g. outgoing 分支）。

不過我不清楚這個分支條件該怎麼設置，所幸搜索到了 Git 設置中 includeIf 的例子。includeIf 與 include 類似，均用以讀取 path 選項指定的外部配置文件，只不過 includeIf 還需要一個條件[^fn:1]，而這裏我的需求是 onbranch。至於 path 選項，在使用相對路徑時是以 include/includeIf 所在配置文件爲參照的。

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

[^fn:1]: 詳細參見 `man 1 git-config` 或 `info "(gitman) git-config"` 中 Conditional includes 部分。