---
title: 【Spring Boot】第12.1課－初探 Spring Security 的認證與授權
date: 2021-06-06 15:48:00
permalink: articles/spring-boot-security-authentication-and-authorization
index_img: /img/index_img/spring-boot.jpg
excerpt: Spring Security 能幫助我們開發有關認證與授權等有關安全管理的功能。本文會進行初始設置，搭配 in-memory 的測試帳號，登入存取 API，藉此建立認證的概念。至於授權，則是透過設計帳號權限，並定義 API 要開放給哪些權限來達成。
categories:
  - ["Spring Boot"]
  - ["Spring Security"]
---

Spring Security 是一套框架，它能幫助我們開發有關認證與授權等有關安全管理的功能。

本文會進行 Spring Security 的初始配置。首先準備簡單的 RESTful API，並搭配 in-memory 的測試帳號，在瀏覽器登入存取 API。藉此讓讀者建立認證的概念。

至於授權的部份，則會設計帳號的權限，並定義 API 要開放給哪些權限。


-----


## 一、前言
Spring Security 是一套可以保護 Spring Boot 應用程式的框架。它扮演著如同保全的角色，能夠管控人員的進出，以及誰可以去什麼地方。

那麼要保護什麼呢？假設我們有一個校務系統，已知有「學生」和「老師」兩種角色。使用情境中，老師可以建立課程資料、給學生打分數；而學生可以選課、給予課程回饋。當然，這些操作都要先登入系統才能進行。

基於這個情境，對學生而言，課程和分數的資料就只能查看，不能建立、修改或刪除。對老師而言，課程回饋只能查看，不能修改。

所以說，Spring Security 這套安全管理的框架，就是要保護服務、資料等各項資源，不會被任意存取。

Spring Security 提供了兩大功能，分別是認證（authentication）與授權（authorization）。認證就相當於前面提到的登入，向系統表示自己是個擁有帳號的使用者。而授權則是系統允許該使用者存取某服務，也就是存取 API。

## 二、程式專案概觀
本系列文章的範例程式，使用的 Spring Boot 版本為 3.4.4。

請建立 Controller，準備一支簡單的 API，做為測試用途。
``` java
@RestController
public class MyController {

    @GetMapping("/home")
    public String home() {
        return "系統首頁";
    }
}
```

啟動程式後，讀者可在瀏覽器上前往 `http://localhost:8080/home`，確認有出現該 API 回傳的「系統首頁」字串。
<img src="{{ permalink }}spring-security-call-api-in-browser.png" />

接著請在 pom.xml 檔案添加 Spring Security 的依賴，並且重新整理，將函式庫下載到程式專案中。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

即便現在我們尚未做任何設定，但 Spring Security 預設將自動保護所有 API。此時重新啟動程式，讀者再前往該網址，便會看見內建的登入畫面。
<img src="{{ permalink }}spring-security-login-page.png" />

雖然在前後端分離的系統中，並不會使用這種登入畫面，然而在初學 Spring Security 時，它是用來測試認證與授權的好管道。

Spring Security 在每次程式啟動時，都會產生一個叫「user」的帳號。而密碼為隨機值，在 console 中可找到如下的訊息：
``` text
Using generated security password: 8698d55e-33dc-461c-80e4-a8e364b23a6c
```

這時在登入畫面輸入該組帳密，就又能看見剛剛的「系統首頁」文字了。

而 Spring Security 也有內建登出畫面，網址為 `http://localhost:8080/logout`。
<img src="{{ permalink }}spring-security-logout-page.png" />

在本文第三節登入測試帳號後，可透過此畫面來登出，以切換不同帳號。

## 三、實作認證功能
### （一）準備測試帳號
實務開發中，都是在資料庫儲存使用者的帳密。然而接下來的範例程式，若馬上引進資料庫，那我們就勢必得先準備資料庫的服務、設計使用者的資料表欄位，並在 Spring Boot 中串接。

為避免讀者在學習上分心，以及文章篇幅過長，筆者在本文會使用 Spring Security 提供的「in-memory user」功能，快速建立簡易的測試帳號。

待本文結束，對 Spring Security 的認證與授權有概念後，在<a href="/articles/spring-boot-security-authentication-integrating-with-mongodb-database/" target="_blank">第 12.2 課</a>會抽換成使用資料庫來儲存帳密。

以下建立了一個配置類別，並冠上 `@EnableWebSecurity` 注解。有關 Spring Security 的各項設定，都能在這裡透過程式碼來自定義。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public InMemoryUserDetailsManager inMemoryUserDetailManager() {
        UserDetails user1 = User
                .withUsername("user1")
                .password("{noop}111")
                .build();
        UserDetails user2 = User
                .withUsername("user2")
                .password("{noop}222")
                .build();
        UserDetails user3 = User
                .withUsername("user3")
                .password("{noop}333")
                .build();
        return new InMemoryUserDetailsManager(List.of(user1, user2, user3));
    }
}
```

上面建立了 `InMemoryUserDetailsManager` 元件，它會被 Spring Security 讀取。其用途顧名思義就是在記憶體中管理帳號，建構子接收了 `UserDetails` 型態的物件。

至於建立帳號的方式，則是呼叫 `User` 類別提供的靜態方法，傳入帳號與密碼。此處建立了三個使用者：
| 帳號 | 密碼 |
|-|-|
|user1|111|
|user2|222|
|user3|333|

重新啟動程式後，讀者可在瀏覽器分別用這些帳號前往 `http://localhost:8080/home`。若成功，則代表這些測試帳號是有用的。

### （二）密碼加密
設定密碼時，在前面加上了 `{noop}` 的字串。原因是配置 in-memory user 的密碼時，需指定密碼的加密演算法。以下舉例其中幾項：
| 前綴 | 演算法 | 密碼原文 | 參數寫法 |
|-|-|-|-|
|{noop}|不加密|123|{noop}123|
|{bcrypt}|BCrypt|456|{bcrypt}$2a$12$YowIkLKzGwPjMt6jtCkvDuCA7Vxb/81pQaJvGgtbKjMgVYtMs2DKK|
|{sha256}|SHA256|password|{sha256}97cde38028ad898ebc02e690819fa220e88c62e0699403e94fff291cfffaf8410849f27605abcbc0|

在實務上，我們會先將密碼的原文加密後，才儲存資料庫。其用意是為了保護客戶的資料，避免資料庫內容外洩，或者員工監守自盜。在範例程式中指定密碼的加密演算法，便是為了模擬出這個情境。

為了方便示範，筆者將以未加密的密碼來儲存。其中「noop」是「No Operation」的意思。若讀者有興趣，[這裡](https://bcrypt-generator.com/)也提供 BCrypt 演算法的轉換與驗證工具。

## 四、實作授權功能
### （一）準備測試用 API
上一節，我們已經做到建立測試帳號，並通過登入畫面的「認證」。不過 Spring Security 還有另一個環節，那就是「授權」。

授權指的是系統允許使用者存取某項服務。換句話說，一個人即便能夠登入，也不代表能使用所有的功能。正如同本文第一節所舉的例子，學生不能編輯課程資料，老師也不能選課。

在 Controller 中，請讀者準備其他 API。連同先前的「系統首頁」，現在共有 5 支 API。
``` java
@RestController
public class MyController {

    @GetMapping("/register")
    public String register() {
        return "註冊畫面";
    }
    
    @GetMapping("/home")
    public String home() {
        return "系統首頁";
    }

    @GetMapping("/selected-courses")
    public String selectedCourses() {
        return "修課清單";
    }

    @GetMapping("/course-feedback")
    public String courseFeedback() {
        return "課程回饋";
    }

    @GetMapping("/members")
    public String members() {
        return "使用者列表";
    }
}
```

在這個例子中，我們希望「註冊畫面」不需登入即可存取；「系統首頁」需登入才能存取；「修課清單」只有學生能存取；「課程回饋」只有老師能存取；「使用者列表」只有管理員能存取。

### （二）添加權限
接著回到 Spring Security 的配置類別，為測試帳號設計「權限」（authority）。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public InMemoryUserDetailsManager inMemoryUserDetailManager() {
        UserDetails user1 = User
                .withUsername("user1")
                .password("{noop}111")
                .authorities("STUDENT")
                .build();
        UserDetails user2 = User
                .withUsername("user2")
                .password("{noop}222")
                .authorities("TEACHER")
                .build();
        UserDetails user3 = User
                .withUsername("user3")
                .password("{noop}333")
                .authorities("ADMIN", "TEACHER")
                .build();
        return new InMemoryUserDetailsManager(List.of(user1, user2, user3));
    }
}
```

呼叫 `authorities` 方法，可以為 in-memory user 添加一至多個權限。至於權限的名稱，則是由我們自己取名，此處包含「學生」、「老師」與「管理員」。其中「user3」這個帳號同時具有老師與管理員權限。

### （三）授權規則
設計好帳號的權限後，接下來要對 Controller 的 API 進行保護，定義它們要開放給具有哪些權限的人存取。

請在 Spring Security 的配置類別，建立 `SecurityFilterChain` 元件。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
        return httpSecurity
                .formLogin(Customizer.withDefaults())
                .authorizeHttpRequests(requests -> requests
                        .requestMatchers(HttpMethod.GET, "/register").permitAll()
                        .requestMatchers(HttpMethod.GET, "/selected-courses").hasAuthority("STUDENT")
                        .requestMatchers(HttpMethod.GET, "/course-feedback").hasAnyAuthority("TEACHER", "ADMIN")
                        .requestMatchers(HttpMethod.GET, "/members").hasAuthority("ADMIN")
                        .anyRequest().authenticated()
                )
                .build();
    }
}
```

Spring Security 會將 `HttpSecurity` 物件注入到建立元件的方法中。透過該物件的一系列方法呼叫，我們能站在安全管理的角度，自定義 request 到達後端時的應對方式。

一開始的 `formLogin` 方法，是啟用先前的登入畫面，便於我們繼續進行測試。

接下來的 `authorizeHttpRequests` 方法，其用途是設定要如何進行授權。定義時，需提供「API」與「授權規則」這兩個部份。

呼叫 `requestMatchers` 方法，可傳入 API 路徑與 HTTP 方法；呼叫 `anyRequests` 方法，代表要對「其餘」的 API 做設定。這是有先後順序之分的，就像 Java 語言的「if → else if → else」，是由上而下逐一判斷。

提供完 API 後，接著要定義授權規則。以下舉例幾個可用的方法：
| 方法名稱 | 意義 |
|-|-|
|permitAll|不必登入認證就能存取。|
|hasAuthority|需具備某一個權限才能存取。|
|hasAnyAuthority|只要具備任一個權限就能存取。|
|authenticated|需登入認證才能存取。|

除了上表列出的方法，讀者也可參考 `access` 方法，搭配 Spring 表達式（Spring Expression Language，SpEL）實現複雜的規則。下面的例子，是只有兼具管理員與老師權限的帳號才能存取 API，用到了「AND」邏輯的語法。
``` java
.access(new WebExpressionAuthorizationManager("hasAuthority('ADMIN') AND hasAuthority('TEACHER')"))
```

撰寫完設定後，讀者可重新啟動程式，在瀏覽器分別使用學生、老師與管理員的帳號存取這 5 支 API，確認是否符合規則。

最後補充，定義 API 授權規則時，路徑的部份除了精確地逐一寫出來，也能透過萬用字元來「模糊匹配」。下表是用法與範例：

| 萬用字元 | 意義 | 範例寫法 | 適用 | 不適用 |
|-|-|-|-|-|
|*|0 到多個字元|/courses/*|/courses、/courses/123|/courses/123/draft|
|**|0 到多個階層|/courses/**|任何「/courses」開頭的路徑|-|
|?|1 個字元|/courses/?|/courses/1|/courses/123、/courses|
|?* 或 *?|1 到多個字元|/courses/?*|/courses/1、/courses/123|/courses|

到目前為止，測試帳號都是使用 in-memory user。下一篇將特別實作自定義的認證方式，抽換成使用資料庫來儲存帳號、密碼與權限。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch12.1-security-authentication-and-authorization)。

下一課：<a href="/articles/spring-boot-security-authentication-integrating-with-mongodb-database/" target="_blank">【Spring Boot】第12.2課－在 Spring Security 整合資料庫進行認證</a>