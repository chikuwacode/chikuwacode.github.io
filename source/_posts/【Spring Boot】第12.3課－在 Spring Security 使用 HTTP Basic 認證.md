---
title: 【Spring Boot】第17.3課－在 Spring Security 使用 HTTP Basic 認證
date: 2024-05-08 14:00:00
permalink: articles/spring-boot-security-http-basic-authentication
index_img: /img/index_img/spring-boot.jpg
excerpt: 在學習 Spring Security 時，經常透過內建的登入畫面來認證。但在前後端分離時，我們需獨立於 Spring Boot 之外來存取 API，也就是透過前端程式或 Postman。本文會介紹 HTTP Basic 認證，在 request header 攜帶帳密，以取代登入畫面。
categories:
  - ["Spring Boot", "Spring Security"]
---

在前兩篇文章，我們都是在 Spring Security 的登入畫面通過認證後，利用瀏覽器存取 Controller 的 GET API。但其他如 POST、PUT 與 DELETE 方法的請求，就無法透過這種方式了。特別是在前後端分離的系統，是完全要獨立於 Spring Boot 來存取。

本文會介紹一種叫做「HTTP Basic」的認證方式，經由在 request header 攜帶帳密的方式，取代在登入畫面的認證。


-----


## 一、準備 RESTful API
本節會在 Controller 設計多支 RESTful API，目的是用來測試不同權限的帳號，能否被 Spring Security 授權存取。列出如下：
|| API || 用途 || 授權對象 ||
|-|-|-|
|GET /home|回傳「系統首頁」字串|所有人，包含尚未通過認證的人|
|GET /courses|回傳「課程列表」字串|所有通過認證的人|
|POST /select-course|回傳「選課成功」字串|學生|
|PUT /courses|回傳「更新課程成功」字串|老師|

範例程式中，使用者將具有「學生」或「老師」其中一個權限，在本文第二節建立測試使用者時可以見到。

以下為 Controller 的程式碼。
``` java
@RestController
public class MyController {

    @GetMapping("/home")
    public String home() {
        return "系統首頁";
    }

    @GetMapping("/courses")
    public String getCourses() {
        return "課程列表";
    }

    @PostMapping("/select-course")
    public String selectCourse() {
        return "選課成功";
    }

    @PutMapping("/courses")
    public String updateCourse() {
        return "更新課程成功";
    }
}
```

為求範例程式簡單，故 API 僅回傳字串。

## 二、配置 Spring Security
### （一）設計 API 授權規則
本節進行 Spring Security 的配置，請確認 pom.xml 檔案已經添加以下依賴。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

以下是進行 Spring Security 相關配置的類別。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
        return httpSecurity
                .authorizeHttpRequests(requests -> requests
                        .requestMatchers(HttpMethod.GET, "/home").permitAll()
                        .requestMatchers(HttpMethod.POST, "/select-course").hasAuthority("STUDENT")
                        .requestMatchers(HttpMethod.PUT, "/courses").hasAuthority("TEACHER")
                        .anyRequest().authenticated()
                )
                .csrf(csrf -> csrf.disable())
                .httpBasic(Customizer.withDefaults())
                .build();
    }
}
```

呼叫 `authorizeHttpRequests` 方法，可進一步設計各個 API 的授權規則。這裡依照本文第一節的說明進行實作。

呼叫 `csrf` 方法，可進一步停用對於 CSRF 攻擊的保護機制，讓 Postman 順利存取 API。停用的理由已在上一篇說明，在此不贅述。

呼叫 `httpBasic` 方法，可啟用「HTTP Basic」的認證方式。我們能在 request header 攜帶帳密，取代在 Spring Security 登入畫面進行認證。

### （二）準備測試使用者
以下準備了 2 個測試使用者。
|| 帳號 || 密碼 || 權限 ||
|-|-|-|
|user1|111|學生|
|user2|222|老師|

以下是透過 `InMemoryUserDetailsManager` 元件，來準備測試使用者。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    // ...

    @Bean
    public UserDetailsService inMemoryUserDetailManager() {
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
        return new InMemoryUserDetailsManager(List.of(user1, user2));
    }
}
```

附帶一提，`InMemoryUserDetailsManager` 本身就有實作 `UserDetailsService` 介面。若讀者想改為串接真實的資料庫，只要基於此介面進行抽換即可，做法可參考<a href="/articles/spring-boot-security-authentication-integrating-with-mongodb-database/" target="_blank">第12.2課</a>。

## 三、HTTP Basic 認證方式
### （一）簡介
在先前的文章中，都是在瀏覽器經由 Spring Security 的登入畫面進行認證，隨後再前往特定網址，存取 Controller 中的 GET API。

但 RESTful API 並非只有 GET 方法而已。因此在前後端分離的開發上，仍然要獨立於 Spring Boot 之外來存取 API，也就是透過前端程式或 Postman 之類的工具。

Spring Security 支援一種叫做「HTTP Basic」的認證方式。其做法是在「Authorization」這個 request header 攜帶帳號與密碼，就像在登入畫面輸入帳密一樣。而後端會在每次接收到 request 時，就先進行認證。

### （二）攜帶 request header
那麼具體上要如何攜帶 header 以存取 API 呢？首先我們要將帳號與密碼的值，轉換為 Base64 編碼。[這裡](https://zh-tw.rakko.tools/tools/24/)提供 Base64 編碼與解碼的工具。

舉例來說，帳號為「user1」，密碼為「111」，則取 `user1:111` 的編碼（以冒號隔開帳密），即 `dXNlcjE6MTEx`。而 `user2:222` 的編碼為 `dXNlcjI6MjIy`。

接著使用 Postman 工具存取 API 時，請切換到「Authorization」頁籤，在下拉式選單選擇「Basic Auth」，並填寫帳號與密碼。
<img src="{{ permalink }}spring-security-postman-basic-auth-interface.png" />

此時再切換到「Headers」頁籤，讀者可看到 Postman 已經幫我們自動添加 Authorization 的 header 了。它的值是「Basic」加一個半形空格，再加 Base64 編碼。
<img src="{{ permalink }}spring-security-postman-basic-auth-header.png" />

而前端程式也是如此，每次存取 API 時，都必須在 header 攜帶這種可以證明自己身份的資料。以生活情境來比喻，就像搭乘公司的電梯，會刷門禁卡來確認身份一樣。

### （三）優缺點
要向後端證明自己的身份，終究是來自帳號與密碼。HTTP Basic 認證的優點就是簡單方便，將帳密做個編碼，放在 request header 即可。

若程式專案採用其他認證方式，勢必有其他前置作業。比方說使用<a href="/articles/spring-boot-security-implement-authentication-filter-with-jwt/" target="_blank">第 12.6 課</a>會介紹的 JWT 認證，需撰寫 JWT 的產生與解析程式；使用 OAuth 認證，則需串接第三方登入。

相對而言，HTTP Basic 適合用於較不嚴謹的場合，例如自我練習、課程作業。或者開發中的新系統已經有雛形，要示範操作功能給別人看，也能考慮暫時使用簡易的 HTTP Basic 認證。

至於缺點，筆者認為有 2 點。第一點是 Base64 的編碼資料，是可以被還原成原文的。因此當 header 的值外洩，盜用者便可一直存取 API，直到密碼被更換。這也意味著，需使用「HTTPS」對整個 request 加密，才更加安全。

第二點是延續前面所述，我們在 header 攜帶的 HTTP Basic 認證資料，是沒有時效性的。與同樣是在 header 攜帶的 JWT 相比，JWT 可設置有效期限。即便洩漏出去，盜用者也只有在一小段時間內（由開發者決定）能用而已。


我們在本文已經脫離瀏覽器來存取 RESTful API。那麼在後端 API 的程式邏輯中，要如何知道是誰在存取呢？下一篇將介紹「Security Context」，向 Spring Security 取得使用者的認證資訊。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch12.3-security-http-basic-authentication)。

上一課：<a href="/articles/spring-boot-security-authentication-integrating-with-mongodb-database/" target="_blank">【Spring Boot】第12.2課－在 Spring Security 整合資料庫進行認證</a>

下一課：<a href="/articles/spring-boot-security-context-authentication-info/" target="_blank">【Spring Boot】第12.4課－從 Security Context 取得 API 存取方的認證資訊</a>