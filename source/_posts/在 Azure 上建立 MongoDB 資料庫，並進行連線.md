---
title: 在 Azure 上建立 MongoDB 資料庫，並進行連線
date: 2024-12-06 14:30:11
permalink: articles/azure-create-cosmos-db-for-mongodb
index_img: /img/index_img/azure.jpg
excerpt: 若能將資料庫放上雲端服務，便能簡化伺服器的管理工作。本文介紹如何在 Azure 上建立 MongoDB 資料庫，並分別在 NoSQLBooster 這款 GUI 工具，以及 Spring Boot 後端程式來連線。
categories:
  - ["Azure"]
  - ["MongoDB"]
---

在軟體開發時，我們會搭配資料庫來儲存資料。以前的傳統做法是下載安裝程式，在電腦上安裝。後來也可選擇利用 Docker，下載 image 並運行 container。

但這兩個方法，都必須準備一台電腦作為伺服器。若能放上雲端服務，便能減少伺服器的管理工作。

本文介紹如何在 Azure 上建立 MongoDB 資料庫，並分別在 NoSQLBooster 這款 GUI 工具，以及 Spring Boot 後端程式來連線。


-----


## 一、建立資料庫
Azure 推出的「Cosmos DB」是一種資料庫服務，它提供了 MongoDB 的 API，讓我們可以用原本操作 MongoDB 的方式來存取。

首先在 Azure 上搜尋「Azure Cosmos DB for MongoDB (vCore)」。進入頁面後，點擊建立。
<img src="{{ permalink }}azure-mongodb-list.png" />

以下是筆者選擇的設定，並提供 MongoDB 的帳號與密碼。其餘則使用預設值，讀者可視自己的情況調整。
* 叢集名稱：vincentdemomongodb
* 免費層級：打勾，免費使用基本功能
* 位置：South India（目前只有這個能選）
* MongoDB 版本：7.0
* 管理使用者名稱：vincent

<img src="{{ permalink }}azure-mongodb-basic-setting.png" />

接著切換到「網路」頁籤，設定防火牆規則。設定的目的，是開放哪些 IP 可以連線到這台資料庫。
<img src="{{ permalink }}azure-mongodb-firewall-rule-setting.png" />

本文為了測試用途，只簡單地選擇「Add 0.0.0.0 - 255.255.255.255」，代表網路上的所有 IP 都可以連進來。此時 Azure 會跳出安全性的警告，我們點擊繼續即可。

最後並按下「檢閱 + 建立」。確認設定後，再按下「建立」，等待 Azure 部署完成。
<img src="{{ permalink }}azure-mongodb-review-and-create.png" />

## 二、基本管理
### （一）查看資訊
前往資源畫面，在「概觀」能看見基本資訊，包含叢集名稱、管理員名稱等。
<img src="{{ permalink }}azure-mongodb-overview.png" />

### （二）可存取的 IP
在「設定」→「網路」的畫面，可找到建立資料庫時所設定的防火牆規則。若日後想更改，可在此進行設定。
<img src="{{ permalink }}azure-mongodb-firewall-rule-setting-2.png" />

### （三）備份與還原
我們可下載 Mongo DB 的[資料庫工具](https://www.mongodb.com/try/download/database-tools)（database tools），在 command line 環境使用指令，進行備份與還原。

備份時，需使用「mongodump」指令：
``` sh
mongodump --uri="mongodb+srv://<username>:<password>@vincentdemomongodb.mongocluster.cosmos.azure.com/<database>?tls=true&authMechanism=SCRAM-SHA-256" --out="<path>"
```

請在 URI 提供連線字串。
* username：管理使用者名稱，如本文的「vincent」
* password：密碼
* database：要備份的資料庫名稱，如 MongoDB 會內建一個資料庫叫「test」

而「out」參數可提供備份檔的儲存位置，如「.\backup」。

還原時，需使用「mongorestore」指令：
``` sh
mongorestore --uri="mongodb+srv://<username>:<password>@vincentdemomongodb.mongocluster.cosmos.azure.com/<database>?tls=true&authMechanism=SCRAM-SHA-256" "<path>"
```

同樣需在 URI 提供連線字串，其中包含要還原到的資料庫。

而最後一個參數為備份檔的來源路徑。延續本文範例，此處的值將會是「.\backup\test」。

## 三、在 NoSQLBooster 連線
本節讓我們使用 [NoSQLBooster](https://nosqlbooster.com/) 這款 GUI 工具，連線到 Azure 上建立好的 Mongo DB 資料庫。

在「設定」→「連接字串」的畫面，可找到應用程式要連結到 MongoDB 的連接字串，它是由叢集名稱和 Azure 的網域所組成。請讀者先複製起來。
<img src="{{ permalink }}azure-mongodb-connection-string.png" />

在 NoSQLBooster 建立新連線的視窗中，請點擊「From URI」，並貼上剛剛的連線字串，別忘了將「`<password>`」字眼替換成自己的密碼。
<img src="{{ permalink }}azure-mongodb-nosqlbooster-from-uri.png" />

接著可點擊「Test Connection」，進行測試。最後為連線取名，建立完成後，便能連上。
<img src="{{ permalink }}azure-mongodb-nosqlbooster-open-connections.png" />

裡面預設會有一個叫做「test」的資料庫。

## 四、在 Spring Boot 連線
本節讓我們在 Spring Boot 後端程式中，針對 Azure 上的 MongoDB 進行配置。

首先在 pom.xml 檔案中，添加 Spring Data MongoDB 的依賴。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

接著在 application.properties 檔案中，提供連線字串。它是從 Azure 上的「設定」→「連接字串」畫面所取得。別忘了將「`<password>`」的字替換成自己的密碼。
```
spring.data.mongodb.uri=mongodb+srv://vincent:<password>@vincentdemomongodb.mongocluster.cosmos.azure.com/?tls=true&authMechanism=SCRAM-SHA-256&retrywrites=false&maxIdleTimeMS=120000

spring.data.mongodb.database=test
```

完成後，啟動 Spring Boot 程式。若 console 沒有出現錯誤訊息，代表成功連線。