---
title: 【Spring Boot】第12.5課－將 Spring Security 與 JWT 結合，實作登入 API
date: 2021-06-06 15:52:00
permalink: articles/spring-boot-security-implement-login-api-with-jwt
index_img: /img/index_img/spring-boot.jpg
excerpt: 為了存取受到 Spring Security 保護的 RESTful API，前端發出請求時，必須出示「Token」，來得到後端的授權。本文首先會介紹 Token 的概念，以及 JWT 的組成。接著利用第三方函式庫，實作出產生與解析 JWT 的程式，藉此設計出登入用途的 API。
categories:
  - ["Spring Boot"]
  - ["Spring Security"]
---

為了存取受到 Spring Security 保護的 RESTful API，前端發出請求時，必須出示某種「證明」，來得到後端的授權。例如先前介紹的 HTTP Basic 認證，會在「Authorization」這個 header 攜帶帳號與密碼的 Base64 編碼。

做為另一個方案，本文將實作叫做「JWT」的資料。首先會介紹 Token 的概念，以及 JWT 的組成。接著利用第三方函式庫，實作出產生與解析 JWT 的程式，藉此設計出登入用途的 API。


-----


## 一、JWT 介紹
### （一）什麼是 Token
讀者是否有在學校圖書館借書時要刷學生證，或者在公司搭電梯要刷門禁卡的經驗？由於這些設施不會開放給外人使用，因此需要出示類似「識別證」的物品，來表明自己是誰。

在 Web API 的範疇，Token 就如同識別證，後端可從中得到該人的身份。筆者在第 17.3 課介紹了 HTTP Basic 認證，我們存取 API 時，就有在「Authorization」這個 header 攜帶帳號與密碼的 Base64 編碼，這就是一種簡易的 Token。

若想實現更強大的身份認證，系統會設計成讓使用者登入後，得到以下 2 種不同用途的 Token，請前端保存在本地。
* Access Token：存取 API 時會攜帶於 request header，證明自己的身份。為了安全性，其有效期限較短，例如 1 小時。因此若外洩，則盜用者也無法使用太久的時間。
* Refresh Token：在換發新的 Access Token 時提供。有效期限較長，例如 7 天。
透過 Refresh Token，就能在不重新登入的條件下，取得新的 Access Token，有助於使用者體驗。等到 Refresh Token 到期，就真的要重新登入，取得以上 2 種 Token 了。

那麼如果 Refresh Token 被盜用怎麼辦？其實絕對安全是很難做到的。Access Token 雖然經常被攜帶，但有效期限短，而 Refresh Token 被攜帶的機會少很多。透過這樣的設計，來降低被盜用的風險與影響程度。

### （二）JWT 的內容
JWT 的全名是「JSON Web Token」，它是一種將 JSON 資料進行編碼的標準，現今經常被當作證明身份的資料。

JWT 是由「標頭」（header）、「內容」（payload）與「簽名」（signature）這 3 個部份組成。因包含簽名，故又稱為 JWS（JSON Web Signature）。

標頭主要包含兩個欄位。「alg」代表簽名時使用的演算法，「typ」代表 token 的種類。
``` json
{
    "alg": "HS256",
    "typ": "JWT"
}
```

我們可以在 JWT 中存放各種資料，以下列舉幾種常見的標準欄位。
|欄位名稱|全稱|意義|
|-|-|-|
|iss|Issuer|JWT 的發行方|
|sub|Subject|這個 JWT 的主體是誰，通常是指使用者的唯一標識|
|iat|Issued At|JWT 的建立時間|
|nbf|Not Before|JWT 的生效時間|
|exp|Expiration Time|JWT 的到期時間|

除了標準欄位，也能存放自定義的欄位。例如使用者的信箱、權限、偏好等。要注意的是，JWT 中的所有資料都會經過 Base64 編碼的處理。然而這種編碼方式是可以被還原成原文的，因此請不要存放私密資料，如密碼或身份證字號等。

筆者撰寫本文時，注意到 ChatGPT 有舉例在自定義欄位存放使用者 id，心想與上述的「sub」欄位非常類似。進一步詢問後，得到的解釋是 sub 欄位可以是使用者在資料庫中的唯一 id，而自定義欄位可提供商業邏輯的 id。

以生活情境來比喻，假設某人大學畢業後，就讀同校的研究所。由於校方已有該人在大學時期的資料，因此資料庫中勢必有個 id 對應到他（如 MySQL 的流水號或 MongoDB 的 ObjectId）。而大學和研究所的「學號」可能會不同，這就屬於商業邏輯的 id。

### （三）JWT 的簽名
透過「簽名」，可以幫助驗證內容是否合法或遭竄改。簽名的產生過程，是將內容經過演算法與密鑰（secret key）的處理所得到。

為方便讀者理解，讓我們以簡單的「凱薩加密法」為例。凱薩加密法的原理，是將欲加密的內容（假設都是英文字母），偏移一定的量。假設偏移量為 3，則「apple」的加密結果為「dssoh」。此時密鑰為「3」，簽名為「dssoh」。

JWT 的內容與簽名是會一併傳遞的。驗證的方式，是將收到的內容，以相同的密鑰與演算法產生另一組簽名，並與收到的簽名比對是否相同。若不同，則代表內容是不合法的。

例如收到的內容是「peach」，而收到的偽造簽名是「qfbdi」（偏移量 1）。此時系統使用偏移量 3 所產生的簽名為「shdfk」。由於兩組簽名不同，故視為認證失敗。同時我們也意會到密鑰是不能外流的。

回到 JWT 的簽名。其產生過程，是將標頭與內容分別進行 Base64 編碼後串接起來，再透過特定演算法與密鑰的處理所產生。

JWT 的官網提供了[工具](https://jwt.io/#debugger-io)，用來產生與驗證 JWT。
<img src="{{ permalink }}jwt-debugger.png" />

整個 JWT 的組成，就是 `標頭的編碼.內容的編碼.簽名`。

## 二、準備程式專案
### （一）Spring Security 配置
認識了 JWT 後，本文讓我們在程式專案中實作出來。首先進行 Spring Security 的配置，請確認 pom.xml 檔案已經添加以下依賴。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

以下是 Spring Security 的配置。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception {
        return httpSecurity
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(requests -> requests.anyRequest().permitAll())
                .build();
    }

    @Bean
    public UserDetailsService inMemoryUserDetailsManager() {
        UserDetails user = User
                .withUsername("user1")
                .password("111")
                .authorities("STUDENT", "ASSISTANT")
                .build();
        return new InMemoryUserDetailsManager(List.of(user));
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
}
```

呼叫 `csrf` 方法，可進一步停用對於 CSRF 攻擊的保護機制，讓 Postman 工具能順利存取 API。

呼叫 `authorizeHttpRequests` 方法，可進一步定義各個 RESTful API 的授權規則。此處設定為 API 不需通過認證也可存取。

另外也準備了測試使用者，帳號為「user1」，密碼為「111」，權限為「學生」與「助理」。另外也需建立 `PasswordEncoder` 元件，以指定密碼的加密方式，筆者選擇不加密。

附帶一提，`InMemoryUserDetailsManager` 本身就有實作 `UserDetailsService` 介面。若讀者想改為串接真實的資料庫，只要基於此介面進行抽換即可，做法可參考<a href="/articles/spring-boot-security-authentication-integrating-with-mongodb-database/" target="_blank">第 12.2 課</a>。

### （二）設計登入 API
接下來讓我們設計一個登入用的 API，它會接收帳號與密碼，並於本文第三節完成後回傳 JWT。以下是用來當作 request body 的類別。
``` java
public class LoginRequest {
    private String username;
    private String password;

    // getter, setter ...
}
```

以下是 Controller 的程式，注入了 `UserDetailsService` 與 `PasswordEncoder` 元件。此處提供了 `POST /login` 這支 API，邏輯很單純地是先查詢使用者，再比對密碼。
``` java
@RestController
public class MyController {

    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @PostMapping("/login")
    public String login(@RequestBody LoginRequest request) {
        UserDetails user = userDetailsService.loadUserByUsername(request.getUsername());
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new BadCredentialsException("Authentication fails because of incorrect password.");
        }

        return "帳密正確，回傳 JWT";
    }
}
```

下圖是使用 Postman 進行測試的結果。
<img src="{{ permalink }}spring-boot-login-api-test.png" />

若帳密有誤，則拋出 Spring Security 內建的 `BadCredentialsException` 例外。

## 三、實作產生 JWT
### （一）程式實作
這節讓我們實作產生 JWT 的程式。在 JWT 官網的 [libraries](https://jwt.io/libraries) 頁面，可找到許多開源的 library。本文挑選「JJWT」，其操作方式簡單，且 GitHub 的星星數最多，相當知名。

根據該 library 的 [GitHub](https://github.com/jwtk/jjwt) 頁面所寫，請在 pom.xml 檔案添加依賴。
``` xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>

<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

以下建立一個叫做「JwtService」的類別，負責 JWT 相關的事務。
``` java
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;

public class JwtService {
    private final SecretKey secretKey;
    private final int validSeconds;

    public JwtService(String secretKeyStr, int validSeconds) {
        this.secretKey = Keys.hmacShaKeyFor(secretKeyStr.getBytes());
        this.validSeconds = validSeconds;
    }

    // TODO
}
```

在範例程式中，除了透過密鑰來簽名，也會設定 JWT 的有效時間。而密鑰的建立方式，是將一組自定義的字串傳入 `Keys.hmacShaKeyFor` 方法所產生，而這是不能外流的。

接著將 JwtService 建立為元件。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public JwtService jwtService(
            @Value("${jwt.secret-key}") String secretKeyStr,
            @Value("${jwt.valid-seconds}") int validSeconds
    ) {
        return new JwtService(secretKeyStr, validSeconds);
    }
}
```

關於用來建立密鑰的字串，以及 JWT 的有效時間，此處取自 application.properties 檔案的參數，因此請在該檔案中定義。
``` properties
jwt.secret-key=1234567890abcdefghij1234567890ab
jwt.valid-seconds=60
```

以上兩個參數，讀者可自行取名和定義值。筆者設計成 JWT 的有效期限為 60 秒。附帶一提，這款 library 規定密鑰字串的長度至少要 256 位元，即 32 個字。

事前準備完成後，便能撰寫程式產生 JWT。以下宣告了一個叫做「createLoginAccessToken」的方法，它會接收 `UserDetails` 介面的物件，將裡面的資料附加到 JWT 中。
``` java
import java.time.Instant;

public class JwtService {
    private final SecretKey secretKey;
    private final int validSeconds;
    // ...

    public String createLoginAccessToken(UserDetails user) {
        // 計算過期時間
        long expirationMillis = Instant.now()
                .plusSeconds(validSeconds)
                .getEpochSecond()
                * 1000;

        // 準備 payload 內容
        Claims claims = Jwts.claims()
                .issuedAt(new Date())
                .expiration(new Date(expirationMillis))
                .add("username", user.getUsername())
                .add("authorities", user.getAuthorities())
                .build();

        // 簽名後產生 JWT
        return Jwts.builder()
                .claims(claims)
                .signWith(secretKey)
                .compact();
    }
}
```

首先計算出 JWT 的過期時間，做法是將現在時間加上有效秒數，再換算為毫秒。

接著呼叫 `Jwts.claims` 方法，逐一添加 payload 的內容。JWT 的標準欄位如「iss」、「exp」等，都有對應的方法能直接呼叫。而自定義的欄位，可呼叫 `add` 方法來提供，此處傳入了 `UserDetails` 物件的帳號與權限。

最後呼叫 `Jwts.builder` 方法，將 payload 傳入。用密鑰簽名後，呼叫 `compact` 方法，終於產生 JWT 的字串。

### （二）在登入 API 應用
完成產生 JWT 的程式後，便可注入到 Controller，應用在登入的 API 了。
``` java
@RestController
public class MyController {
    // ...
    
    @Autowired
    private JwtService jwtService;

    @PostMapping("/login")
    public String login(@RequestBody LoginRequest request) {
        UserDetails user = userDetailsService.loadUserByUsername(request.getUsername());
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new BadCredentialsException("Authentication fails because of incorrect password.");
        }

        return jwtService.createLoginAccessToken(user);
    }
}
```

在此將通過身份認證的 `UserDetails` 傳入 JwtService 中，產生 JWT 字串後回傳。

下圖是使用 Postman 進行測試的結果。
<img src="{{ permalink }}spring-boot-login-api-return-jwt.png" />

範例為了方便，故以純文字回傳 JWT。若讀者想以 JSON 格式的 response body 回傳，也是可以的。

## 四、實作解析 JWT
### （一）程式實作
本節會實作解析 JWT 內容的程式。除了在本文確認 JWT 的內容是否與我們預期的相同，也為下一篇將解析出的使用者資料放入「Security Context」來做準備。
``` java
public class JwtService {
    private final SecretKey secretKey;
    private final int validSeconds;
    private final JwtParser jwtParser;

    public JwtService(String secretKeyStr, int validSeconds) {
        this.secretKey = Keys.hmacShaKeyFor(secretKeyStr.getBytes());
        this.jwtParser = Jwts.parser().verifyWith(secretKey).build();
        this.validSeconds = validSeconds;
    }

    // ...

    public Claims parseToken(String jwt) throws JwtException {
        return jwtParser.parseSignedClaims(jwt).getPayload();
    }
}
```

在 JwtService 中，透過本文第三節建立的 `SecretKey` 密鑰物件，可再建立出 `JwtParser` 物件。它能夠解析 JWT 字串，並比對簽名來驗證是否合法。

範例程式中宣告了叫做「parseToken」的方法，它的用途是接收 JWT 字串，並使用 `JwtParser` 進行解析。若解析失敗，例如過期、不合法，或格式錯誤，則拋出 `JwtException`。

事實上，`JwtException` 並非 checked expection。筆者是希望呼叫這個方法的其他地方，都能留意這項例外的處理，所以才定義在方法上。

最後可得到 Claims 介面的物件，它就是在本文第三節產生 JWT 時，我們放入 payload 的內容。此外，該介面本身正好有繼承 `Map<String, Object>` 介面。

### （二）在 Controller 使用
為了實際確認解析的結果，讓我們回到 Controller 提供另一支 API。它會從「Authorization」這個 request header 讀取 JWT，並交由 JwtService 解析後回傳結果。
``` java
@RestController
public class MyController {
    private static final String BEARER_PREFIX = "Bearer ";

    @Autowired
    private JwtService jwtService;

    // ...

    @GetMapping("/who-am-i")
    public Map<String, Object> whoAmI(@RequestHeader(HttpHeaders.AUTHORIZATION) String authorization) {
        String jwt = authorization.substring(BEARER_PREFIX.length());
        try {
            return jwtService.parseToken(jwt);
        } catch (JwtException e) {
            throw new BadCredentialsException(e.getMessage(), e);
        }
    }
}
```

在 request header 攜帶 JWT 時，格式是先以「Bearer」加一個半形空格做為前綴，才加上 JWT。因此後端接收到 header 後，要先排除該前綴，才能進行解析。

若解析發生問題，則對 `JwtException` 進行例外處理。此處再拋出 `BadCredentialsException`，讓 Spring Security 回傳 HTTP 401 的狀態碼。

下圖是使用 Postman 進行測試的結果。
<img src="{{ permalink }}spring-boot-api-parse-jwt.png" />

操作方式是切換到 Authorization 頁籤，在下拉式選單選取「Bearer Token」，並填入 JWT。發送 request 時，Postman 會自動在「Authorization」這個 header 加上「Bearer」與一個半形空格的前綴，此為攜帶 Token 的格式。

本文到目前為止，已經完成 JWT 的產生與解析。後續小節將基於目前實作出的程式做優化。

## 五、讓登入 API 回傳更多資料
根據筆者在前公司的經驗，登入 API 除了回傳 JWT，還會附上其他資料，例如使用者的 id、名字、偏好設定等。目的是控制前端的行為，例如登入後要前往什麼畫面、如何顯示某些 HTML，或者將其攜帶於 query string，有各種用途。

本節讓我們在登入 API 多回傳一點資料。以下是用來當作 response body 的類別，叫做「LoginResponse」。
``` java
public class LoginResponse {
    private String jwt;
    private String username;
    private List<String> authorities;

    public static LoginResponse of(String jwt, UserDetails user) {
        LoginResponse res = new LoginResponse();
        res.jwt = jwt;
        res.username = user.getUsername();
        res.authorities = user.getAuthorities()
                .stream()
                .map(GrantedAuthority::getAuthority)
                .toList();

        return res;
    }
    
    // getter ...
}
```

接下來調整 Controller 的程式，將 `UserDetails` 物件中的帳號與權限資料，放入 LoginResponse 物件後回傳。
``` java
@RestController
public class MyController {
    // ...

    @PostMapping("/login")
    public LoginResponse login(@RequestBody LoginRequest request) {
        UserDetails user = userDetailsService.loadUserByUsername(request.getUsername());
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new BadCredentialsException("Authentication fails because of incorrect password.");
        }

        String jwt = jwtService.createLoginAccessToken(user);

        return LoginResponse.of(jwt, user);
    }
}
```

下圖是使用 Postman 進行測試的結果。
<img src="{{ permalink }}spring-boot-login-api-return-json.png" />

然而在本節的範例中，能夠從 `UserDetails` 物件取得的內容，就只有帳號與權限而已。若讀者想在 JWT 本身或登入 API 的 response body 攜帶更多使用者資料，不妨搭配實作自己的 `UserDetails` 和 `UserDetailsService`。詳情可參考<a href="/articles/spring-boot-security-context-authentication-info/" target="_blank">第 12.4 課</a>。

筆者在文末附上的完成後專案，也會融入這部份的實作。下圖是使用 Postman 測試登入 API 的結果。
<img src="{{ permalink }}spring-boot-login-api-return-more-data.png" />

例子中可看到額外回傳了使用者的 id 與暱稱。

## 六、使用 AuthenticationManager 元件進行認證
還記得 `UserDetails` 介面所提供的方法，除了回傳帳號、密碼與權限，還包含一些狀態嗎？
|方法名稱|意義|
|-|-|
|isEnabled|帳號已啟用|
|isAccountNonLocked|帳號未被鎖定|
|isAccountNonExpired|帳號未過期|
|isCredentialsNonExpired|密碼未過期|

Spring Security 進行身份認證時，會一併檢查這 4 種狀態。若其中一個方法回傳 false，即便帳號與密碼正確，仍應視為認證失敗。

本文剛開始實作登入 API 時，我們只是很簡單地在 Controller 呼叫 `UserDetailsService` 與 `PasswordEncoder` 元件來確認帳密。若還想檢查狀態，勢必會寫出更多程式碼。事實上，可以請 Spring Security 幫忙做這些檢查工作。

在<a href="/articles/spring-boot-security-context-authentication-info/" target="_blank">第 12.4 課</a>，筆者介紹了 HTTP Basic 認證的原始碼，其中提到了介面為 `AuthenticationManager` 的物件。它的用途便是接收帳號與密碼，並在底層使用 `UserDetailsService` 與 `PasswordEncoder` 進行身份認證。此外也會檢查上述的帳號狀態。

本節的目的，就是要替換成使用 `AuthenticationManager` 來認證。然而 Spring Security 並不會在 IoC 容器自動建立它的元件，我們得自行建立。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public AuthenticationManager authenticationManager(HttpSecurity httpSecurity) throws Exception {
        return httpSecurity.getSharedObject(AuthenticationManagerBuilder.class).build();
    }
}
```

回到 Controller，我們將帳號與密碼的值，包裝成 `UsernamePasswordAuthenticationToken` 物件，再傳入 `AuthenticationManager.authenticate` 方法進行認證。
``` java
@RestController
public class MyController {

    @Autowired
    private AuthenticationManager authenticationManager;

    // ...

    @PostMapping("/login")
    public String login(@RequestBody LoginRequest request) {
        Authentication token = new UsernamePasswordAuthenticationToken(
            request.getUsername(),
            request.getPassword()
        );
        Authentication auth = authenticationManager.authenticate(auth);
        UserDetails user = (UserDetails) auth.getPrincipal();

        return jwtService.createAccessToken(user);
    }
}
```

認證成功後，會得到新的 `Authentication` 物件，其 `principal` 欄位為 `UserDetails` 介面的物件。將它取出運用，就能接續原本的程式碼了。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch12.5-security-implement-login-api-with-jwt)。

上一課：<a href="/articles/spring-boot-security-context-authentication-info/" target="_blank">【Spring Boot】第12.4課－從 Security Context 取得 API 存取方的認證資訊</a>

下一課：<a href="/articles/spring-boot-security-implement-authentication-filter-with-jwt/" target="_blank">【Spring Boot】第12.6課－實作 Spring Security 的認證 Filter（以 JWT 為例）</a>