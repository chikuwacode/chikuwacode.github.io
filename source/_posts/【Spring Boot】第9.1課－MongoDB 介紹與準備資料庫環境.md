---
title: 【Spring Boot】第9.1課－MongoDB 介紹與準備資料庫環境
date: 2019-06-23 08:33:39
permalink: articles/spring-boot-mongodb-introduction-and-setup
index_img: /img/index_img/spring-boot.jpg
excerpt: 後端程式會透過資料庫來儲存與查詢資料。本文將介紹 MongoDB 資料庫，並在 Spring Boot 專案中進行連接。接著介紹基本語法，讓讀者知道 MongoDB 的資料與語法，是以 JSON 格式來表示。由於 JSON 格式可直接與程式物件的資料做對應，因此適合那些希望能靈活調整資料欄位的應用程式。
categories:
  - ["Spring Boot"]
---

後端程式需要透過資料庫來儲存與查詢資料。在先前的文章，範例程式都是以 Java 的 Map 資料結構來暫存，一旦重新啟動程式，資料便消失了。

在第 9 課系列中，本文首先會介紹 MongoDB 這款資料庫，並在 Spring Boot 中確認可以連接上。接著介紹基本語法，讓讀者在後續的練習中，知道如何在資料庫中查看資料，以便確認結果。


-----


## 一、MongoDB 介紹
### （一）什麼是 NoSQL
提到資料庫，讀者可能會先聯想到如 MySQL、SQL Server 與 Oracle 等常見的關聯式資料庫。這種資料庫，會透過資料表設計來定義資料規格。也透過約束（constraint）來要求不同表的資料必須能保持關聯。

與之相對的是「NoSQL」（Not Only SQL），使用上不需要嚴格地去設計資料表和約束關係。以下列出幾種常見的 NoSQL，它們都有各自適用的場景。
* MongoDB：使用 JSON 格式儲存資料，故內容可直接與程式物件對應。適合希望能隨著需求，靈活調整資料欄位的系統。此外也易於進行「反正規化」，若設計得當，有助於應用程式更快地得到查詢結果。
* Redis：會以「鍵值對」（key-value pair）的形式，來儲存每一筆資料。由於運行時，資料會載入到記憶體中，因此適合作為快取的用途。
* Elasticsearch：同樣使用 JSON 格式儲存資料。本身是一種「全文檢索引擎」，除了一般查詢，更大的特色是提供關鍵字查詢與相關度評分的功能。適合用來開發站內搜尋與推薦系統。

### （二）集合與文件
在關聯式資料庫中，會使用「資料表」（table）來儲存資料，而一筆資料稱為「資料列」（row）。

在 MongoDB 中，則使用其他名詞來表達這兩個概念。「集合」（collection）對應到資料表，而「文件」（document）對應到資料列。換句話說，文件會以 JSON 格式記錄資料內容，而集合會儲存許多文件。

MongoDB 會為每一筆文件產生叫做「_id」的欄位，它相當於資料的主鍵，不會有重複值。其資料型態預設為特有的「Object Id」，值由 24 個 0～9 與 a～f 字元組成。值的產生與當下時間有關，因此有遞增的現象。

後續在使用 Spring Boot 串接 MongoDB 時，文件還會被自動加入叫做「_class」的欄位，它的值會是 Java 類別的 package 路徑。這是為了在程式查詢資料時，能夠將文件內容轉換成正確的 Java 物件。

## 二、準備 MongoDB 環境
### （一）Docker 容器
讀者可在命令列（command line）環境下，透過 Docker 指令啟動 MongoDB 的服務。
``` sh
docker run -d --name "MongoDB_4.4.29" -p 27017:27017 mongo:4.4.29
```

以上取用的 MongoDB 版本為 4.4.29。容器取名為「MongoDB_4.4.29」，運行在 27017 的 port 號上。

其他版本可參考 [Docker Hub](https://hub.docker.com/_/mongo)。關於 Docker 的操作，本文將不詳細介紹。

### （二）圖形化介面工具
為了直接查詢資料庫中的資料，或者進行寫入，讀者可安裝圖形化介面（GUI）工具。本文選擇「[NoSQL Booster](https://nosqlbooster.com/downloads)」。

開啟 GUI 後，點擊左上方的「Connect」按鈕，會出現管理資料庫連線設定的視窗。若讀者想存取不同伺服器上的資料庫，就能在這裡建立多個需要的連線（connection）。
<img src="{{ permalink }}nosqlbooster-connections.png" />

按下「Create」按鈕，可填寫設定並建立一個連線。
<img src="{{ permalink }}nosqlbooster-connection-editor-basic.png" />

在上圖中，「Server」欄位填寫的是資料庫的位址與 port 號。而「Name」欄位是為這個連線命名，方便我們在 GUI 辨識。

最後按下「Save & Connect」按鈕，就能連線到資料庫。

### （三）建立資料庫
按下 GUI 上方的「SQL」按鈕，可開啟查詢視窗。下圖撰寫了兩句語法。
<img src="{{ permalink }}nosqlbooster-use-query-to-create-database-collection.png" />

使用 `use`，可切換到指定的資料庫，即便不存在也可切換。此處選擇叫做「school」的資料庫。

使用 `db.createCollection`，可在資料庫中建立集合（collection），此處建立叫做「student」的集合。
``` javascript
use "school"
db.createCollection("student");
```

按下「Run」按鈕，就會執行視窗中的全部指令。由於 school 這個資料庫中開始有內容了，此時它才會被建立出來。

以上這些操作，其實不透過指令，也能藉由 GUI 完成，讀者可自行探索。

## 三、準備程式專案
本節讓我們在 Spring Boot 專案中串接資料庫。請在 pom.xml 檔案添加 MongoDB 的依賴。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

接著在專案的「src\main\resources」路徑下，找到「application.properties」配置檔，並在裡頭添加連線字串。以下是在本地連線到叫做「school」的資料庫。
``` properties
spring.data.mongodb.uri=mongodb://localhost:27017/school
```

若讀者未來要在 Docker 容器中運行 Spring Boot，可能需將網域從「localhost」改為「host.docker.internal」，如下：
``` properties
spring.data.mongodb.uri=mongodb://host.docker.internal:27017/school
```

MongoDB 的 library 會從配置檔中讀取指定 key 的設定值，做為連線字串。

最後請啟動 Spring Boot。若 console 沒有出現 exception 訊息，代表有成功連線到 MongoDB 的服務。若連線不到，可能會看到如下的訊息。
``` text
com.mongodb.MongoSocketOpenException: Exception opening socket
...
Caused by: java.net.ConnectException: Connection refused: no further information
...
```

## 四、基本語法介紹
為了讓讀者可以透過查詢，直接在 GUI 上確認當前資料庫中的資料，本節將介紹一些語法。在撰寫指令操作 MongoDB 時，絕大多數都是使用 Javascript 語言。

以下是插入多筆資料到叫做「student」的集合。而資料庫會自動產生「_id」欄位的值。
``` javascript
db.student.insertMany(
    [
        {
            "name": "Vincent",
            "majority": "資訊管理",
            "contact": {
                "email": "vincent@school.com",
                "phone": "0911111111"
            }
        },
        {
            "name": "Dora",
            "majority": "財務金融",
            "contact": {
                "email": "dora@school.com",
                "phone": "0922222222"
            }
        }
    ]
);
```

以下是查詢集合中的所有資料。在大括號 `{ }` 中會寫上查詢條件，此處為無條件。
``` javascript
db.student.find({});
```

以下是查詢「_id」欄位值為某個 Object Id 的資料。
``` javascript
db.student.find(
    {
        "_id": ObjectId("6640d57d4825f6a549d85c08")
    }
);
```

以下是查詢「majority」欄位值，等於其中一個陣列元素的資料。
``` javascript
db.student.find(
    {
        "majority": {
            "$in": ["資訊管理", "會計"]
        }
    }
);
```

以下是查詢「contact」物件欄位中的「email」欄位，是某個值的資料。
``` javascript
db.student.find(
    {
        "contact.email": "vincent@school.com"
    }
);
```

以下是刪除集合中的所有資料。
``` javascript
db.student.deleteMany({});
```

本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.1-mongodb-introduction-and-setup)。