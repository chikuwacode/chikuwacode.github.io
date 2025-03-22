---
title: 【Spring Boot】第1課－從環境準備、建立專案、打包到啟動程式
date: 2019-05-13 16:41:13
permalink: articles/spring-boot-create-project
index_img: /img/index_img/spring-boot.jpg
excerpt: 一套資訊系統，需要前端與後端整合才完整。這系列的文章，會引進筆者大學畢業以來的工作經驗和自學成果，教導讀者使用 Java 程式語言撰寫 Spring Boot 後端程式。本文會建立一個可以啟動，但還沒有任何功能的 Spring Boot 專案，並使用 Maven 打包成 JAR 檔執行。
categories:
  - ["Spring Boot"]
---

在網路上或補習班，能找到大量手機 App 或網頁等前端開發課程。至今前端開發仍然是想轉職軟體工程師的人的首選。

但一套資訊系統，需要前端與後端結合才完整。這系列的文章，會引進筆者大學畢業以來的工作經驗和自學成果，教導讀者使用 Java 程式語言撰寫 Spring Boot 後端程式。

本文會建立一個可以啟動，但還沒有任何功能的 Spring Boot 專案，並使用 Maven 工具打包成 JAR 檔執行。


-----


## 一、前端與後端的概念
本文開頭所提到的 App 與 Web，會與使用者直接互動，在此將它們歸類於「前端」（front end）。前端會向伺服器發出網路連線請求（request），得到伺服器回覆的結果後，前端程式就進行應變，像是呈現資料或提示訊息。至於這邊的「伺服器」，則屬於後端（back end）的範疇。

我們學習程式語言時，會在開發工具上（如 Eclipse、IntelliJ IDEA）寫一些語法。接著按下執行，跑完之後，程式就結束了。但後端程式不一樣，它啟動後就一直在電腦上待命，並不會自己結束。

後端程式一收到前端的請求，就開始執行資料處理，通常還會伴隨資料庫的存取。最後將處理結果回應（response）給前端。

以生活情境來比喻，就像是我們去餐廳吃飯，跟店員點餐後，廚師就會開始製作餐點。餐點做好後，店員再送到我們面前。此時，我們是使用者，店員是前端，廚師是後端。店員送菜單給廚師，而廚師遞出餐點，就是前端與後端之間的互動。

本文介紹的「Spring Boot」，是一種後端程式的框架。所謂的框架（framework），是將複雜程式的實作細節，封裝起來後的產物。若讀者寫過前端的 Angular、Vue 或手機的 Android 程式，它們也都是一種框架。我們需要根據它們提供的程式物件或方法來開發。

## 二、環境準備
想跟著本系列文章學習 Spring Boot 開發，得先準備好需要的環境與工具。請讀者事先上網下載安裝。

* [Java 17](https://www.azul.com/downloads/?package=jdk)：Spring Boot 目前最新已來到第 3 版，其要求 Java 17 以上的版本。使用 Java 時，需注意授權問題。此連結是由 Azul 提供的 Zulu JDK，可商業使用。
* [Maven](https://maven.apache.org/download.cgi)：這是一種專案的管理工具。用途是在程式專案中引進第三方函式庫。最後進行建置，打包成像是 JAR 檔的執行檔。
* [IntelliJ IDEA](https://www.jetbrains.com/idea/download)：此為開發工具，分為社群版（community）與高級版（ultimate）。後者提供一些較強大的功能，可試用 30 天，而筆者選用免費的社群版。

最後別忘了將 Java 與 Maven 安裝位置的「bin」資料夾，添加到環境變數。可在命令列分別執行 `java --version` 與 `mvn --version`，確認有顯示出版本資訊。
<img src="{{ permalink }}check-java-and-maven-version-by-command.png" />

題外話，下載 Java 與 Maven 時，也能像筆者一樣，透過「SDKMAN」這項工具來輔助。它可幫助我們在電腦上安裝同一套件的不同版本，並輕鬆地切換。有興趣的話，請參考「[Guide to SDKMAN!](https://www.baeldung.com/java-sdkman-intro)」文章。

## 三、建立 Spring Boot 專案
Spring Boot 官方在網路上提供了實用的工具，叫做「[Spring Initializr](https://start.spring.io/)」。它能夠幫助我們輕鬆地產生初始的程式專案。
<img src="{{ permalink }}spring-initializr-example.png" />

根據畫面上的項目，可以先勾選「Maven Project」、「Java 17 版本」、「Spring Boot 3.x.x 版本」與「打包成 JAR 檔」。

而「Group」、「Artifact」等區塊，可填寫足以代表公司的網域及產品名稱。其他欄位用預設值就好。

最後是圖中右方的「Dependencies」區塊，能在此加入自己已知需要的依賴，也就是前面提到的第三方函式庫。我們預先選擇「Spring Web」，因為會利用它來開發「Web API」，並啟動內建的 Tomcat 伺服器軟體，做為 Spring Boot 程式運行的載體。

完成後按下「GENERATE」，便能將專案下載回來。無論是自己練習，或是去公司工作，在開發前都會先取得像這樣的程式專案資料夾。

打開 IntelliJ 後，請在入口畫面點擊「Open」，並選擇該專案的目錄。
<img src="{{ permalink }}intellij-start-page.png" />

稍等一段時間，開發工具便完成開啟專案的動作了。

## 四、專案概觀
### （一）pom.xml 檔案
在專案中，讀者能找到「pom.xml」檔案，全名為「Project Object Model」。它是 Maven 專案的必要檔案，紀錄了一些描述和使用的第三方函式庫等。先前在 Spring Initializr 填寫的東西，可在此處這邊找到。
<img src="{{ permalink }}pom-file-init.png" />

其中剛剛的「Spring Web」函式庫就在圖中下方的 `<dependencies>` 標籤中。若讀者日後需要其他函式庫，亦可在此添加。

若更動了 pom.xml 檔，IntelliJ 右下方會出現「Maven projects need to be imported」的訊息。屆時請按下「Import Changes」，讓開發工具重新整理，或把函式庫下載回來。

### （二）application.properties 配置檔
簡單來說，這份檔案可以存放一些「設定值」，例如資料庫的 IP 位址，或串接服務的帳號密碼等。這樣一來，我們就不必硬寫（hard code）在程式碼中。在<a href="/articles/spring-boot-application-properties-configuration/" target="_blank">第 7 課</a>會專門介紹。

另外，Spring Boot 本身也有自己的設定值。例如「server.port」可指定要運行在哪個埠號（port），預設是 8080。
``` properties
server.port=8080
```

### （三）啟動類別
接著請讀者看到「DemoApplication.java」檔案，裡面有個 `main` 方法，代表這是專案的啟動類別。筆者習慣將此類別改名為「Application」。類別上冠有 `@SpringBootApplication` 注解，代表這是啟動 Spring Boot 的類別。而 `SpringApplication.run` 方法則是進行啟動的動作。
<img src="{{ permalink }}spring-boot-entrypoint-class.png" />

按下程式碼行數那邊的綠色箭頭就能選擇「Run 'Application.main( )'」來啟動程式。從下方的 Console 視窗會看見一些訊息。
<img src="{{ permalink }}spring-boot-start-console.png" />

首先是 Spring 的圖案，下方寫著版本。接著是第一行，提到了 Java 的版本。第三行的「Tomcat initialized with port(s): 8080 (http)」，則代表啟動時使用了內建的 Tomcat 伺服器軟體。

而最後一行的「Started Application in XXX seconds…」，代表已經啟動完畢，可以接受請求。

## 五、打包專案
將開發成果交付出去後，若要正式運行後端程式，並不會是開啟 IntelliJ，按 Run 之後擺著。本節讓我們將打包出一個 JAR 檔，這樣在任何有安裝 Java 的環境（包含 Linux 系統），都可以啟動它。

請先在命令列環境（如 Windows 的命令提示字元）切換到專案的根目錄，也就是 pom.xml 檔的位置。接著執行 `mvn clean package` 指令，即可開始打包，稍等一段時間就完成了。

事實上，IntelliJ 本身也有自帶終端機（Terminal）的視窗，讀者可直接使用。

上述的 Maven 指令有 `clean` 與 `package` 這兩項操作，它們的目的如下。
* `package`：將專案中的 .java 檔，編譯成 .class 檔。接著連同靜態資源（如圖片檔）與配置檔（如 application.properties），一同打包進 JAR 檔。這些產出會放在叫做「target」的資料夾內。
* `clean`：將 target 資料夾清空，避免比較舊的檔案被打包進 JAR 檔，導致運行起來發生非預期的情況。

讀者可在專案的 target 資料夾中找到一個 JAR 檔。若要執行它，請在切換到 JAR 檔的所在位置，執行 `java -jar {JAR檔名稱}` 指令，如下圖。
<img src="{{ permalink }}spring-boot-execute-jar-file.png" />

在命令列上出現的訊息，與先前在 IntelliJ 上執行是相同的。若要停止程式，按下「Ctrl + C」即可。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch01-create-project)。

下一課：<a href="/articles/spring-boot-restful-api/" target="_blank">【Spring Boot】第2課－認識 RESTful API 與前後端的資料交換</a>