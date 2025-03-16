---
title: 【Spring Boot】第6課－在 application.properties 配置檔提供設定值（以 Java Mail 為例）
date: 2020-02-19 14:29:57
permalink: articles/spring-boot-application-properties-configuration
index_img: /img/index_img/spring-boot.jpg
excerpt: 在專案中串接資料庫或第三方服務，通常都需要那些平台的帳密或其他設定值。我們可以將這些設定值集中寫在一個叫「配置檔」的地方，再由程式去引用，以便根據不同環境彈性切換。本文會以使用 Java Mail 寄信為例子，示範如何配置設定值，並利用 Profile 的功能來切換。
categories:
  - ["Spring Boot"]
---

在串接資料庫或其他第三方服務時，通常都需要那些平台的伺服器位址、帳密，或其他設定值，我們合稱為「連線字串」（connection string）。

若我們希望在不同環境啟動程式時（如測試環境、正式環境），能彈性切換不同的連線字串，那麼可以將這些值寫在一個叫「配置檔」的地方，再由程式去取用。

本文會以使用透過 Java Mail 寄送郵件為例，示範如何配置設定值，並利用 Profile 的功能來切換。最後講解在啟動 JAR 檔時，如何引入配置檔。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch06-start-application-properties-configuration)。

## 一、準備程式專案
### （一）範例專案介紹
在本文的練習用專案，能找到筆者事先準備好的範例程式。

第一個是「MailController」，裡面有一支 RESTful API。我們預期它會接收 request，並執行一段寄送 email 的程式。
``` java
@RestController
public class MailController {

    @PostMapping("/mail")
    public ResponseEntity<Void> sendPlainText(@RequestBody SendMailRequest request) {
        // TODO
        return ResponseEntity.noContent().build();
    }
}
```

第二個是「SendMailRequest」，它會在 controller 接收 request body，包含 email 的收件人、主旨與內容。
``` java
public class SendMailRequest {
    private String[] receivers = new String[0];
    private String subject = "";
    private String content = "";
    
    // getter, setter ...
}
```

### （二）匯入函式庫
請在 pom.xml 檔案添加「Spring Mail」的依賴。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

這項依賴其實是 Spring Boot 對 Java Mail 進行封裝後的產物，讓我們容易使用。

## 二、取得 Google 應用程式密碼
由於本文會搭配郵件服務來做示範，因此需要做一些事前準備。畢竟在程式中串接外部服務時，通常都需要那個平台的帳密。

筆者會使用 Gmail 來寄信，然而 Google 為了安全性考量，不允許我們直接在程式中使用 Google 密碼。因此接下來請依照步驟，產生「應用程式密碼」。

在 Google 帳戶的首頁，前往「安全性」的頁籤，找到「登入 Google 的方式」，進入「兩步驟驗證」畫面。
<img src="{{ permalink }}google-account-security-tab.png" />

從畫面最下方，進入「應用程式密碼」畫面。
<img src="{{ permalink }}google-two-step-verification-page.png" />

在畫面中為這次要建立的密碼取個名字。
<img src="{{ permalink }}google-application-password-setup.png" />

按下「建立」後，即可得到 16 個字的應用程式密碼。
<img src="{{ permalink }}google-application-password-fin.jpg" />

請讀者將這組密碼記在別的地方，因為離開此畫面後，便無法再看到了。

## 三、認識 Spring Mail 的用法
本節會展示如何使用 Spring Mail 發送純文字郵件，藉此讓讀者知道使用 Gmail 寄信時，需要哪些設定值。到了第四節，我們再調整成使用「配置檔」的做法。
``` java
@RestController
public class MailController {

    @PostMapping("/mail")
    public ResponseEntity<Void> sendPlainText(@RequestBody SendMailRequest request) {
        JavaMailSenderImpl sender = createMailSender();

        SimpleMailMessage msg = new SimpleMailMessage();
        msg.setFrom(String.format("%s<%s>", "Spring Mail", sender.getUsername()));
        msg.setTo(request.getReceivers());
        msg.setSubject(request.getSubject());
        msg.setText(request.getContent());

        sender.send(msg);

        return ResponseEntity.noContent().build();
    }

    private JavaMailSenderImpl createMailSender() {
        JavaMailSenderImpl sender = new JavaMailSenderImpl();

        // 郵件服務主機
        sender.setHost("smtp.gmail.com");
        sender.setPort(587);

        // 郵件服務帳密
        sender.setUsername("your_gmail");
        sender.setPassword("your_application_password");

        Properties props = sender.getJavaMailProperties();
        props.put("mail.smtp.auth", true); // 是否向郵件服務驗證身份
        props.put("mail.smtp.starttls.enable", true); // 是否啟用 TLS（傳輸層安全），對通訊加密
        props.put("mail.transport.protocol", "smtp"); // 傳輸協定

        return sender;
    }
}
```

上面的範例程式中，在「createMailSender」方法建立了 `JavaMailSenderImpl` 物件。它需要一些有關郵件服務的參數。例如主機、port 號、帳密等。

而 API 的處理方法「sendPlainText」，則建立了 `SimpleMailMessage` 物件，並填寫郵件的寄件人、收件人、主旨與內容，最後寄出。

此處呼叫 `setFrom` 方法，在寄件人給予 `名稱<信箱>` 格式的字串。這是為了讓收件人能看到我們自定義的寄件人名稱，而不是一個 email 地址。

完成後，讀者可用 Postman 之類的工具呼叫該 API。
``` text
POST http://localhost:8080/mail
{
    "receivers": ["foo@gmail.com", "bar@gmail.com"],
    "subject": "Hello",
    "content": "World"
}
```

確認一下目前的程式能夠寄信到指定的 email 地址。

## 四、開始使用配置檔
### （一）撰寫參數
在第三節，我們把寄信的相關設定值寫死在程式碼中（hard code），這樣的做法可能有一些缺點：
* 當各種服務的連線字串，都散落在專案的不同地方，以後若想查看或修改，就不容易找到當初是寫在哪裡。
* 承上，即使將設定值集中寫在一個 Java 類別作為常數，但每次修改後，都得重新打包出 JAR 檔，才能交付。
* 為了便於管理，我們可以將設定值集中寫在一個叫做「application.properties」的配置檔中。該檔案預設是位於「src\main\resources」資料夾下。若讀者沒有該檔案，請自行建立一個。

接著將寄信時所用到的設定值，以 `key=value` 的格式寫在裡面，以下是一個範例：
``` properties
mail.host=smtp.gmail.com
mail.port=587
mail.username=your_gmail
mail.password=your_application_password
mail.transport-protocol=smtp
mail.smtp.auth=true
mail.starttls.enable=true
mail.display-name=Spring Mail
```

此處的 key 都是筆者自己取名，且單字可用「.」符號來表示階層。

### （二）讀取配置檔參數
將設定值都寫到 application.properties 配置檔後，我們就能在元件中透過 `@Value` 注解，來讀取它們。
``` java
@RestController
public class MailController {

    @Value("${mail.host}")
    private String host;

    @Value("${mail.port}")
    private int port;

    @Value("${mail.username}")
    private String username;

    @Value("${mail.password}")
    private String password;

    @Value("${mail.transport-protocol}")
    private String protocol;

    @Value("${mail.smtp.auth}")
    private boolean smtpAuth;

    @Value("${mail.starttls.enable}")
    private boolean enableStartTls;

    @Value("${mail.display-name:Test Mail}")
    private String displayName;

    @PostMapping("/mail")
    public ResponseEntity<Void> sendPlainText(@RequestBody SendMailRequest request) {
        // ...        
        msg.setFrom(String.format("%s<%s>", displayName, sender.getUsername()));
        // ...
    }

    private JavaMailSenderImpl createMailSender() {
        JavaMailSenderImpl sender = new JavaMailSenderImpl();

        sender.setHost(host);
        sender.setPort(port);
        sender.setUsername(username);
        sender.setPassword(password);

        Properties props = sender.getJavaMailProperties();
        props.put("mail.smtp.auth", smtpAuth);
        props.put("mail.smtp.starttls.enable", enableStartTls);
        props.put("mail.transport.protocol", protocol);

        return sender;
    }
}
```

使用 `@Value` 注解時，需傳入配置檔的 key 名稱。若有需要，讀者也可提供預設值。格式為 `${key 名稱:預設值}`。

### （三）封裝到配置類別
也許讀者已經感受到了，當類別中存在太多 `@Value` 注解，整個類別就會很冗長。

因此，我們也可以考慮將讀取設定值的部份，轉移到另一個元件類別。
``` java
@Configuration
public class MailConfig {

    @Value("${mail.host}")
    private String host;

    @Value("${mail.port}")
    private int port;

    @Value("${mail.username}")
    private String username;

    @Value("${mail.password}")
    private String password;

    @Value("${mail.transport-protocol}")
    private String protocol;

    @Value("${mail.smtp.auth}")
    private boolean smtpAuth;

    @Value("${mail.starttls.enable}")
    private boolean enableStartTls;

    @Value("${mail.display-name:Test Mail}")
    private String displayName;

    // getter ...
}
```

以上建立一個叫做「MailConfig」的類別，並冠上 `@Configuration` 注解，代表這是與配置有關的元件類別。另外也提供 getter 方法供外部呼叫。

之後便可將這個元件注入到需要的地方。
``` java
@RestController
public class MailController {
    
    @Autowired
    private MailConfig mailConfig;

    @PostMapping("/mail")
    public ResponseEntity<Void> sendPlainText(@RequestBody SendMailRequest request) {
        // ...
        msg.setFrom(String.format("%s<%s>", mailConfig.getDisplayName(), sender.getUsername()));
        // ...
    }

    private JavaMailSenderImpl createMailSender() {
        JavaMailSenderImpl sender = new JavaMailSenderImpl();
        
        sender.setHost(mailConfig.getHost());
        sender.setPort(mailConfig.getPort());
        sender.setUsername(mailConfig.getUsername());
        sender.setPassword(mailConfig.getPassword());

        Properties props = sender.getJavaMailProperties();
        props.put("mail.smtp.auth", mailConfig.isSmtpAuth());
        props.put("mail.smtp.starttls.enable", mailConfig.isEnableStartTls());
        props.put("mail.transport.protocol", mailConfig.getProtocol());

        return sender;
    }
}
```

## 五、使用函式庫所指定的設定值
Spring Mail 函式庫本身有提供「自動配置」的功能。也就是說，Spring Mail 自己也有一個類似第四節的 MailConfig 配置類別。它會讀取配置檔中指定 key 名稱的設定值，私下建立出 `JavaMailSenderImpl` 元件。

這麼一來，我們其實直接取用它所建立出來的元件即可，完全不需要自己寫程式來建立。

本節讓我們稍作調整，將配置檔的設定值，改為 Spring Mail 指定的 key 名稱。
``` properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your_gmail
spring.mail.password=your_application_password
spring.mail.properties.mail.transport.protocol=smtp
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

mail.display-name=Spring Mail
```

到這邊為止，除了「mail.display-name」的 key 名稱是筆者自己取名，其餘都是 Spring Mail 特別指定的。

接下來，請在程式中放心地注入 `JavaMailSenderImpl` 元件。
``` java
@RestController
public class MailController {

    @Autowired
    private JavaMailSenderImpl sender;

    @Autowired
    private MailConfig mailConfig;

    @PostMapping("/mail")
    public ResponseEntity<Void> sendPlainText(@RequestBody SendMailRequest request) {
        // ...
        msg.setFrom(String.format("%s<%s>", mailConfig.getDisplayName(), sender.getUsername()));
        // ...

        sender.send(msg);

        return ResponseEntity.noContent().build();
    }
}
```
而先前用來手動建立 `JavaMailSenderImpl` 物件的 createMailSender 方法，則可以移除了。

## 六、使用 Profile 切換配置參數
### （一）部署的環境
根據筆者的工作經驗，寫好的程式至少會在以下環境運行。
* 開發環境：在開發期間，讓工程師確認實作出的功能是否如預期。
* 測試環境：讓測試人員驗證功能是否正常。
* 正式環境：提供服務給真實使用者。

每個環境所需要的設定值可能不同。最典型的例子就是不同環境所使用的資料庫位址和帳密，都不一樣。

針對這個問題，我們可以準備多個配置檔，將不同環境的設定值寫在裡面。並搭配 Spring Boot 提供的「Profile」功能，切換使用不同配置檔。

### （二）切換配置
以下準備了 2 份配置檔，分別用於開發和測試環境。檔名需遵循 application-環境名稱.properties 的格式。

「application-dev.properties」檔案：
``` properties
mail.display-name=Spring Mail (dev)
```

「application-test.properties」檔案：
``` properties
mail.display-name=Spring Mail (test)
```

它們都只有一個叫做「mail.display-name」的設定值，會做為寄件者的名稱。

接著在原本的 application.properties 配置檔中，添加 `spring.profiles.active` 的設定值，就能指定啟動程式時要用哪一份配置檔了。
``` properties
spring.profiles.active=dev

# 其他參數 ...
```

此處的設定值為「dev」，因此 Spring Boot 便會去讀取「application-dev.properties」檔案。至於其他未寫在該配置檔的設定值，則會回頭採用 application.properties。

## 七、啟動 JAR 檔時指定配置檔
第六節的做法適合在開發工具中，切換不同的配置。而本節將說明如何在啟動 JAR 檔時，指定配置檔。

### （一）指定 Profile
前面所建立的那些配置檔，會連同程式專案一起被打包進 JAR 檔。因此在 command line 執行 JAR 檔時，可提供 `-Dspring.profiles.active` 這項參數，指定要採用的配置檔。
``` sh
java -Dspring.profiles.active=test -jar demo.jar
```

此處指定「test」這個 profile，代表要採用「application-test.properties」配置檔。

### （二）自行提供配置檔
若讀者自己有準備一份配置檔，也可於啟動 JAR 檔時進行指定。

請在 JAR 檔的所在位置，建立一個叫做「config」的資料夾，並將配置檔放到裡面。相對位置如下：
``` text
|_ demo.jar
|_ config
    |_ application.properties
```

啟動時，只要簡單地執行以下指令即可，Spring Boot 將會優先讀取 config 資料夾的配置檔。
``` sh
java -jar demo.jar
```

## 八、YAML 檔
YAML 檔是一種常被用來配置設定值的檔案格式，副檔名為「yml」。我們可將 application.properties 檔案，改寫為「application.yml」檔案。Spring Boot 支援以 YAML 檔做為設定值的來源。
``` yml
server:
  port: 8080

spring:
  profiles:
    active: dev
  mail:
    host: smtp.gmail.com
    port: 587
    username: your_gmail
    password: your_application_password
    properties:
      mail:
        transport:
          protocol: smtp
        smtp:
          auth: true
          starttls:
            enable: true

mail:
  display-name: Spring Mail
```

從排版格式便可清楚地看出階層之分。YAML 檔以換行加兩個半形空格縮排為一個階層，相當於 properties 檔的「.」。YAML 格式有時具有更好的可讀性。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch06-fin-application-properties-configuration)。

上一課：<a href="/articles/spring-boot-bean-ioc-di-and-swap" target="_blank">【Spring Boot】第5課－元件的控制反轉、依賴注入與抽換</a>

下一課：<a href="/articles/spring-boot-construct-bean-programmatically" target="_blank">【Spring Boot】第7課－手動進行元件的初始化</a>