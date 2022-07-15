+++
title = "On Misskey"
author = ["Hilton Chain"]
date = 2022-06-26T23:25:00+08:00
tags = ["Misskey", "Fediverse", "SNS", "Docker", "PostgreSQL", "100DaysToOffload"]
categories = ["notes"]
draft = true
image = "cover.png"
+++

[Misskey](https://misskey-hub.net/) 是一個去中心化的 SNS 應用，其使用 [ActivityPub](https://www.w3.org/TR/activitypub/) 協議通訊，已部署的應用稱爲實例。衆多 Misskey 實例與其他使用 ActivityPub 或兼容協議的應用的實例，一同組成相互聯通的 [Fediverse](https://en.wikipedia.org/wiki/Fediverse)。

我在半年前部署了一個 Misskey 實例，現時（2022 年 6 月）就我的體驗評價一下客戶端（Misskey 自帶的網頁前端）。

1.  Pros：
    -   UI 清新，這也是當時吸引我的主要原因。
    -   交互方式有趣，實現了一個堆疊式窗口管理器。
    -   奇奇怪怪的小功能。
    -   簡易的雲存儲實現，媒體文件管理能力尚可（e.g. 無文件格式限制、可引用已有的媒體文件、簡易的文件夾功能）
    -   DSL：MFM（Misskey Flavored Markdown，用於寫作帖文）、AiScript（由 JavaScript 解釋）
    -   主題以及插件（AiScript）系統。
2.  Cons
    -   現時足夠可用的客戶端只有自帶的網頁前端，但後端沒有提供存儲客戶端設置的功能，作爲瀏覽器應用而言比較頭疼。
    -   設置項較爲繁雜。要想輕車熟路地找到特定項目可能得適應幾個月……
    -   除窗口管理器外，功能普遍不夠健壯，與主業務集成不強，同時整個 Misskey 也完全稱不上通用意義上的平臺。
    -   雲存儲功能只能算是能夠簡單管理文件的文件列表。
    -   沒人能保證你順利升級。

總結下來就是： ~~能用。~~ 基本的 SNS 功能可用，附加的各種功能可有可無，有成爲通用平臺的潛力但前提是重構，存儲方面的功能相對糟心。最後，最令人沮喪的一點，我不懂 web 開發。

因此，現時如果想要參與 Fediverse 的基建，我並不推薦 Misskey，但這並不代表我推薦其他應用。

下面的部分是與部署 Misskey 相關的，包含全新部署（Docker 方案）以及數據庫遷移/升級相關的說明。


## 全新部署 {#全新部署}

前提：[Docker](https://docs.docker.com/engine/install/) 和 [docker-compose](https://docs.docker.com/compose/install/)

假定 /srv/misskey 爲 git 倉庫存放路徑、暴露服務於 127.0.0.1:3000，其餘設置項爲 Misskey git 倉庫內示例值。

準備本地倉庫（不打算自己構建鏡像的話可以省掉這步，但仍需保證之後的三個文件存在且位置正確）：

```shell
git clone --branch master https://github.com/misskey-dev/misskey /srv/misskey

cd /srv/misskey
cp .config/{docker_example,docker}.env # 用於設置數據庫的環境變量
cp .config/{example,default}.yml       # Misskey 配置文件
```

此後所有操作均在 /srv/misskey 中進行。


### 數據庫環境變量 {#數據庫環境變量}

```cfg
# /srv/misskey/.config/docker.env

# db settings
POSTGRES_PASSWORD=example-misskey-pass
POSTGRES_USER=example-misskey-user
POSTGRES_DB=misskey
```


### Misskey 配置文件 {#misskey-配置文件}

下面是些需要注意的地方，文件比較長就放 diff 了，當然配置文件還是得完整過一遍。

```diff
# /srv/misskey/.config/default.yml

diff --git a/example.yml b/default.yml
index ef91c86..dbb5330 100644
--- a/example.yml
+++ b/default.yml
@@ -34,7 +34,7 @@ port: 3000
#───┘ PostgreSQL configuration └────────────────────────────────

db:
-  host: localhost
+  host: db
port: 5432

# Database name
@@ -55,11 +55,11 @@ db:
#───┘ Redis configuration └─────────────────────────────────────

redis:
-  host: localhost
+  host: redis
port: 6379
#pass: example-pass
#prefix: example-prefix
-  #db: 1
+  db: 1

#   ┌─────────────────────────────┐
#───┘ Elasticsearch configuration └─────────────────────────────
@@ -111,7 +111,7 @@ id: 'aid'
# inboxJobMaxAttempts: 8

# IP address family used for outgoing request (ipv4, ipv6 or dual)
-#outgoingAddressFamily: ipv4
+outgoingAddressFamily: dual

# Syslog option
#syslog:
@@ -135,10 +135,10 @@ id: 'aid'
#mediaProxy: https://example.com/proxy

# Proxy remote files (default: false)
-#proxyRemoteFiles: true
+proxyRemoteFiles: true

# Sign to ActivityPub GET request (default: false)
-#signToActivityPubGet: true
+signToActivityPubGet: true

#allowedPrivateNetworks: [
#  '127.0.0.1/32'
```


### 容器編排 {#容器編排}

Misskey 倉庫內是一份 v3 的 [docker-compose.yml](https://docs.docker.com/compose/compose-file/compose-file-v3/)，這裏用 Compose specification 的 [compose.yaml](https://docs.docker.com/compose/compose-file/)，這個文件優先級更高，所以也不需要額外的操作。

```yaml
# /srv/misskey/compose.yaml
services:
  web:
    image: misskey/misskey:latest
    build: .
    restart: unless-stopped
    links:
      - db
      - redis
    ports:
      - "127.0.0.1:3000:3000"
    networks:
      - internal_network
      - external_network
    volumes:
      - ./files:/misskey/files
      - ./.config:/misskey/.config:ro

  redis:
    restart: unless-stopped
    image: redis:alpine
    networks:
      - internal_network
    volumes:
      - ./redis:/data

  db:
    restart: unless-stopped
    image: postgres:14-alpine
    networks:
      - internal_network
    env_file:
      - .config/docker.env
    volumes:
      - ./db:/var/lib/postgresql/data

networks:
  internal_network:
    internal: true
  external_network:
```

設置好以後就可以構建鏡像並啓動實例了

```shell
docker-compose up -d --build

# 下面是使用預構建鏡像的情況
docker-compose pull
docker-compose up -d
```

更新實例：

```shell
git pull origin master
docker-compose up -d --build

# 下面是使用預構建鏡像的情況
docker-compose pull
docker-compose up -d
```


## 數據庫遷移 {#數據庫遷移}

PostgreSQL 大版本間互不相容，所以無論是遷移還是升級都得人工轉移數據。

[如前所述](#全新部署)，使用 Misskey git 倉庫內的[示例值](#數據庫環境變量)。

先停止服務：

```shell
docker-compose down
```

然後導出數據到 /srv/misskey/dump.sql，此過程可能會相當耗時。

```shell
docker-compose up -d db
docker-compose exec db pg_dumpall -U example-misskey-user > dump.sql
```

接着準備導入的環境，PostgreSQL 只會在無數據庫文件時初始化。

```shell
dokcer-compose down
mv db db-backup
```

對於升級的情況，現在修改 compose.yaml 中 PostgreSQL 鏡像的版本號（e.g. 將 `postgres:12-alpine` 改爲 `postgres:14-alpine` ）。

然後導入數據，該操作可能較爲耗時。

```shell
docker-compose up -d db
mv dump.sql db/
docker-compose exec db sh -c "psql -U example-misskey-user -d misskey < /var/lib/postgresql/data/dump.sql"
```

導入完成後，可以刪除原先的文件了。

```shell
docker-compose down
rm -r db-backup db/dump.sql
```

爲使用密碼驗證，將 db/pg_hba.conf 末行修改如下：

```cfg
host all all all password
```

完。

> 題圖于壬寅年五月廿八截自[此帖](https://misskey.io/notes/8vuev7ce1i)。