---
title: 在 Azure 上建立 MySQL 資料庫，並進行連線
date: 2024-12-06 14:14:13
permalink: articles/azure-create-mysql-flexible-server
index_img: /img/index_img/azure.jpg
excerpt: 若能將資料庫放上雲端服務，便能簡化伺服器的管理工作。本文介紹如何在 Azure 上建立 MySQL 資料庫，並分別在 Workbench 這款 GUI 工具，以及 Spring Boot 後端程式來連線。
categories:
  - ["Azure"]
  - ["MySQL"]
---
在軟體開發時，我們會搭配資料庫來儲存資料。以前的傳統做法是下載安裝程式，在電腦上安裝。後來也可選擇利用 Docker，下載 image 並運行 container。

但這兩個方法，都必須準備一台電腦作為伺服器。若能放上雲端服務，便能減少伺服器的管理工作。

本文介紹如何在 Azure 上建立 MySQL 資料庫，並分別在 Workbench 這款 GUI 工具，以及 Spring Boot 後端程式來連線。


-----


## 一、建立資料庫
首先在 Azure 上搜尋「適用於 MySQL 彈性伺服器的 Azure 資料庫」。進入頁面後，點擊建立。
<img src="{{ permalink }}azure-mysql-list.png" />

選擇「彈性伺服器」的「進階建立」，可看到比較多的設定選項。
<img src="{{ permalink }}azure-mysql-choose-flexible-server.png" />

以下是筆者選擇的設定，並提供 MySQL 的帳號與密碼。其餘則使用預設值，讀者可視自己的情況調整。
* 伺服器名稱：vincentdemomysql
* 區域：East Asia
* MySQL 版本：8.0
* 工作負載類型：適用於開發或嗜好專案
* 驗證方法：只有 MySQL 驗證
* 管理使用者名稱：vincent

<img src="{{ permalink }}azure-mysql-basic-setting.png" />

接著切換到「網路」頁籤，設定防火牆規則。設定的目的，是開放哪些 IP 可以連線到這台資料庫。
<img src="{{ permalink }}azure-mysql-firewall-rule-setting.png" />

本文為了測試用途，只簡單地選擇「新增 0.0.0.0 - 255.255.255.255」，代表網路上的所有 IP 都可以連進來。此時 Azure 會跳出安全性的警告，我們點擊繼續即可。

最後並按下「檢閱 + 建立」。確認設定後，再按下「建立」，等待 Azure 部署完成。
<img src="{{ permalink }}azure-mysql-review-and-create.png" />

雖然 Azure 會顯示 MySQL 彈性伺服器的預估使用成本，但[官方說明](https://learn.microsoft.com/zh-tw/azure/mysql/flexible-server/how-to-deploy-on-azure-free-account)有提到，只要使用免費帳戶，且使用量在每月限制內，就不需要支付費用。

## 二、基本管理
前往 MySQL 的資源畫面，在「概觀」能看見基本資訊，包含伺服器名稱、管理員名稱等。
<img src="{{ permalink }}azure-mysql-overview.png" />

在「設定」→「資料庫」的畫面，可建立與刪除資料庫。此處筆者建立一個叫「demo」的資料庫。
<img src="{{ permalink }}azure-mysql-create-database.png" />

在「設定」→「備份與還原」的畫面，可確認資料庫的備份及其時間點。Azure 每天會自動備份一次，我們也可手動立即備份。
<img src="{{ permalink }}azure-mysql-backup.png" />

在「設定」→「網路」的畫面，可找到建立資料庫時所設定的防火牆規則。若日後想更改，可在此進行設定。
<img src="{{ permalink }}azure-mysql-firewall-rule-setting-2.png" />

在「設定」→「連線」的畫面，可看見關於連線至資料庫的相關說明。例如從本機的 command line 使用 MySQL 指令進行連線、匯入與匯出資料。另外也有使用 Workbench 進行連線的步驟。
<img src="{{ permalink }}azure-mysql-connection.png" />

## 三、在 Workbench 連線
本節讓我們使用 [Workbench](https://dev.mysql.com/downloads/workbench/) 這款 GUI 工具，連線到 Azure 上建立好的 MySQL 資料庫。

在建立新連線的視窗中，請於 Hostname 填寫第二節在「概觀」中看到的伺服器名稱。在 Username 填寫伺服器系統管理員登入名稱。而 Password 則點擊「Store in Valut」後，再填寫密碼。
<img src="{{ permalink }}azure-mysql-workbench-setup-new-connection.png" />

接著可點擊「Test Connection」，進行測試。最後建立完成後，點擊該連線便可進入。

進入後，讀者就能在左方看見先前建立的「demo」資料庫。
<img src="{{ permalink }}mysql-workbench-schema-list.png" />

後續也能順利進行 CRUD 操作。

## 四、在 Spring Boot 連線
本節讓我們在 Spring Boot 後端程式中，針對 Azure 上的 MySQL 資料庫進行配置。

首先在 pom.xml 檔案中，添加依賴，包含 Spring Data JPA 框架和 MySQL 驅動程式。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

接著在 application.properties 檔案中，提供 Spring Data JPA 的相關配置。
```
spring.datasource.url=jdbc:mysql://vincentdemomysql.mysql.database.azure.com:3306/demo
spring.datasource.username=vincent
spring.datasource.password=your_password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.properties.hibernate.dialect_version=8
spring.jpa.properties.hibernate.dialect.storage_engine=innodb

spring.jpa.hibernate.ddl-auto=update

spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
```

由於本文是介紹連線到 Azure 上的資料庫，因此請讀者留意前三項配置，也就是連線字串中的伺服器名稱、資料庫名稱，以及帳密。

完成後，啟動 Spring Boot 程式。若 console 沒有出現錯誤訊息，代表成功連線。

後續也能使用 Spring Data JPA 提供的注解（annotation），進行資料表設計。
``` java
@Entity
@Table(name = "student")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", length = 30, unique = true, nullable = false)
    private String name;

    @Column(name = "grade")
    private int grade;

    // getter, setter ...
}
```

重新啟動 Spring Boot 後，讀者可回到 Workbench 確認產生後的資料表。