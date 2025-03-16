---
title: 【Spring Boot】第9.1課－準備 MySQL 資料庫與認識 Spring Data JPA
date: 2024-05-29 16:00:00
permalink: articles/spring-boot-setup-mysql-and-introduce-jpa
index_img: /img/index_img/spring-boot.jpg
excerpt: MySQL 資料庫是一種關聯式資料庫，與其他 NoSQL 相比，關聯是資料庫是職缺中更常見的要求。本文會啟動 MySQL 的服務，並在 Spring Boot 專案中配置各種設定值，確認可以連線上。最後介紹「Spring Data JPA」這套框架，了解它的由來。
categories:
  - ["Spring Boot"]
---

在<a href="/articles/spring-boot-mongodb-introduction-and-setup" target="_blank">第 8 課</a>系列，我們了解如何在 Spring Boot 操作 MongoDB。而第 9 課系列要學習的是操作 MySQL 資料庫，它是一種關聯式資料庫，具有嚴謹的性質。它除了是一般課程常見的題材，幾乎也是職缺的必備要求。

本文會啟動 MySQL 的服務，並在 Spring Boot 專案中配置各種參數，確認可以連線上。最後介紹「Spring Data JPA」這套框架，了解它的由來。


-----


## 一、準備 MySQL 環境
### （一）Docker 容器
讀者可在命令列（command line）環境下，透過 Docker 指令啟動 MySQL 的服務。
``` sh
docker run -d --name "MySQL_8.2.0" -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mongo:8.2.0
```

以上取用的 MySQL 版本為 8.2.0。容器取名為「MySQL_8.2.0」，運行在 3306 的 port 號上。而 root 帳號的密碼設為「123456」。

其他版本可參考 [Docker Hub](https://hub.docker.com/_/mysql)。關於 Docker 的操作，本文將不詳細介紹。

### （二）圖形化介面工具
為了在後續的練習中能夠查詢資料庫，直接確認裡面的資料，讀者可安裝圖形化介面（GUI）工具。本文選擇「[Workbench](https://dev.mysql.com/downloads/workbench/)」。

開啟 GUI 後，點擊「MySQL Connections」字樣旁邊的「+」號，會出現資料庫連線設定的視窗。若讀者想存取不同伺服器上的資料庫，就能在這裡建立多個需要的連線。
<img src="{{ permalink }}mysql-workbench-setup-new-connection.png" />

在上圖中，「Connection name」欄位是為這個連線命名，方便我們在 GUI 辨識。其他欄位則填寫資料庫的位址、port 號與帳號。至於密碼，需點擊「Store in Vault …」後，在小視窗輸入。

最後按下「OK」，即可建立連線。

### （三）建立資料庫
按下 GUI 上方的「SQL +」按鈕，可開啟查詢視窗。下圖撰寫了兩句語法，第一句是建立名稱為「school」的資料庫，第二句是切換過去。
<img src="{{ permalink }}mysql-workbench-use-query-to-create-database.png" />
``` sql
CREATE DATABASE `school`;
USE `school`;
```

按下閃電符號後，就會執行視窗中的指令。以上這些操作，其實不透過指令，也能藉由 GUI 完成，讀者可自行探索。

## 二、準備程式專案
### （一）添加依賴
本節讓我們在 Spring Boot 專案中串接資料庫。請在 pom.xml 檔案添加「Spring Data JPA」與「MySQL Connector」的依賴。
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

如果讀者有用過 JDBC（Java Database Connectivity），應該對存取資料庫的程式流程還留有一點印象，大致包含以下步驟：
1. 建立資料庫連線。
2. 撰寫原生 SQL 語法，產生「Statement」物件。
3. 將裝有查詢結果的「Result Set」物件，轉換成商業邏輯要用的程式物件。
4. 釋放資源。

由於太過繁瑣，於是「Spring Data JPA」這套框架封裝了此過程，讓開發者能更輕鬆地存取資料庫。

事實上，Spring Data JPA 或 JDBC 並非只能用來存取 MySQL 而已。其他常見的關聯式資料庫，如 SQL Server、Oracle 與 PostgreSQL 也都支援，因為這些資料庫廠商提供了「驅動程式」（Driver）。

### （二）配置參數
請在專案的「src\main\resources」路徑下，找到一個叫做「application.properties」的配置檔，並在裡頭撰寫設定值。
``` properties
# 資料來源
spring.datasource.url=jdbc:mysql://localhost:3306/school
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# 資料庫資訊
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.properties.hibernate.dialect_version=8
spring.jpa.properties.hibernate.dialect.storage_engine=innodb

# 資料表配置
spring.jpa.hibernate.ddl-auto=update

# 是否在 console 印出 SQL 指令並對其格式化
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.format_sql=true
```

首先提供資料來源（data source），包含 MySQL 的位址、帳密，以及驅動程式的 package 路徑。此處連線到叫做「school」的資料庫。而驅動程式是來自於前面添加的「mysql-connector-j」依賴。

若讀者未來要在 Docker 容器中運行 Spring Boot，可能需將網域從「localhost」改為「host.docker.internal」，如下：
``` properties
spring.datasource.url=jdbc:mysql://host.docker.internal:3306/school
```

Spring Data JPA 預設是先封裝另一個叫做「Hibernate」的框架，而它又封裝了 JDBC。Hibernate 提供了與資料庫互動的強大功能，本身可獨立使用。Spring Data JPA 只是將其整合進來，讓我們容易應用於 Spring Boot 程式中罷了。

為了讓 Hibernate 根據不同的資料庫執行正確的指令，我們需要提供代表資料庫種類的「方言」，以及資料庫的版本與儲存引擎。此處提供 `MySQLDialect` 方言、版本 8 與 InnoDB 儲存引擎。

Spring Data JPA 允許我們在程式中直接定義資料表（table），並於啟動 Spring Boot 程式時，讓 Hibernate 自動去資料庫中做配置。有以下幾個選項。
* create：重新建立 table，故原有的資料會遺失，請慎用。
* create-drop：同 create，且程式關閉後，會自動刪除 table。
* update：會根據我們的定義去更新 table，補上新欄位，但不刪除舊欄位。
* validate：會檢查 table 是否缺少我們在程式中定義的欄位。若有，則拋出例外，具有提醒的效果。
* none：不做自動配置。

最後請啟動 Spring Boot。若 console 沒有出現 exception 訊息，代表有成功連線到 MySQL 的服務。

這份叫做 application.properties 的配置檔，在<a href="/articles/spring-boot-application-properties-configuration" target="_blank">第 6 課</a>有專門介紹。讀者只要知道 Spring Data JPA 會從中讀取指定名稱的設定值即可。

### 三、定義資料的實體類別
在資料庫中，每張 table 在程式中都對應到一個實體（entity）類別。比方說想儲存學生資料，那就在程式中設計對應的類別，如下：
``` java
import jakarta.persistence.*;

@Entity
@Table(name = "student")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private int grade;
    
    // getter, setter ...
}
```

該類別叫做「Student」，包含 id、名字與年級，共 3 個欄位。

使用 `@Entity` 注解，代表它是要儲存到資料庫的實體類別。而 `@Table` 注解，則設定它在資料庫中，要對應到叫做「student」的 table 。

在 id 欄位，冠上了 `@Id` 注解，代表它是 table 中的主鍵（Primary Key，PK）。而 `@GeneratedValue` 注解，則設定主鍵值的產生方式。由於 MySQL 支援自增長（auto increment）的流水號 id，因此選擇 `GenerationType.IDENTITY`。

重新啟動 Spring Boot，讀者便可在 GUI 中看到建立好的 table 了。
<img src="{{ permalink }}mysql-workbench-describe-table-with-few-fields.png" />

## 四、Spring Data JPA 介紹
在本文第二節，我們添加了「Spring Data JPA」的依賴，本節將進一步介紹。

### （一）JDBC 使用範例
當 JDBC 結合對應的驅動程式，便可存取 MySQL 等關聯式資料庫，以下為示意程式：
``` java
public class Main {
    private static final String url = "jdbc:mysql://localhost:3306/school";
    private static final String username = "root";
    private static final String password = "123456";

    public static void main(String[] args) {
        List<Student> students = new ArrayList<>();
        
        Connection connection = null;
        Statement statement = null;
        ResultSet resultSet = null;
        try {
            // 建立連線，用原生語法查詢
            connection = DriverManager.getConnection(url, username, password);
            statement = connection.createStatement();
            resultSet = statement.executeQuery("SELECT * FROM `student`;");
            
            // 轉換成商業邏輯物件
            while (resultSet.next()) {
                Long id = resultSet.getInt("id");
                String name = resultSet.getString("name");
                int grade = resultSet.getInt("grade");
                students.add(new Student(id, name, grade));
            }
        } catch (Exception e) {
            // ...
        } finally {
            // 釋放 Connection、Statement、ResultSet 的資源 ...
        }
        
        // 運用商業邏輯物件 ...
    }
}
```

由於使用程式碼操作的過程太繁瑣，因此有些框架將它封裝起來。

### （二）物件關聯對映（ORM）
示意程式中，有個環節是將商業邏輯物件與資料庫的資料互相轉換。比方說查詢時，需取出結果中的每個欄位值，在程式中建立出物件；插入或更新時，則需拼湊出 SQL 語法。

因此，「物件關聯對映」（Object-Relational Mapping，ORM）這項技術出現了，為的就是封裝這段轉換過程。其中 Hibernate、MyBatis、EclipseLink 等 ORM 框架就有提供這項功能，可單獨引進使用。

這些 ORM 框架，並不是只有提供物件和資料之間的轉換功能而已，它們本身就是對整個使用 JDBC 的過程進行封裝。

### （三）Spring Data JPA 框架
Spring 為了讓我們在程式中更容易存取資料庫，於是推出 Spring Data JPA，將 ORM 框架整合進來。但它只整合那些有實現 JPA（Java Persistance API）規範的 ORM 框架，如 Hibernate、EclipseLink。

所謂的 JPA 規範，其實我們在本文第三節建立 Student 類別時已經接觸到了，例如「jakarta.persistence」套件下的 `@Entity`、`@Table` 與 `@Id` 等注解。後續文章也會使用 `@Column` 注解，進行欄位的細部設定；或透過 `@OneToOne` 注解，設計一對一關聯。

Spring Data JPA 預設採用 Hibernate 框架來存取資料庫，讓我們在操作上，能夠以 Java 程式語言為導向，不必依賴於不同資料庫種類的原生 SQL 語法。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.1-setup-mysql-and-introduce-jpa)。

上一課：<a href="/articles/spring-boot-mongo-repository-customize-query" target="_blank">【Spring Boot】第8.3課－在 MongoRepository 定義查詢條件與排序方式</a>

下一課：<a href="/articles/spring-boot-mysql-column-definition-with-jpa" target="_blank">【Spring Boot】第9.2課－使用 JPA 設計實體類別與 MySQL 資料表欄位</a>