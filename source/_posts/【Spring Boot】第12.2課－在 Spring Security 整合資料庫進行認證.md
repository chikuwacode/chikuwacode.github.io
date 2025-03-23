---
title: 【Spring Boot】第12.2課－在 Spring Security 整合資料庫進行認證
date: 2023-11-03 13:21:00
permalink: articles/spring-boot-security-authentication-integrating-with-mongodb-database
index_img: /img/index_img/spring-boot.jpg
excerpt: 實務開發中，我們是透過資料庫來儲存使用者的帳號與密碼。本文首先會將專案串接上資料庫。接著向讀者介紹 Spring Security 認證的相關介面，並實作自定義的認證程式。藉此將資料庫所儲存的帳號、密碼與權限資料，整合到認證流程中。最後介紹加密密碼的方式，以提高安全性。
categories:
  - ["Spring Boot"]
  - ["Spring Security"]
---

在上一篇，我們建立了 Spring Security 認證與授權的概念，並利用 in-memory user 的功能準備測試帳號。然而實務開發中，是透過資料庫來儲存使用者的帳號與密碼。

本文首先會將專案串接上資料庫。接著向讀者介紹 Spring Security 認證的相關介面，並實作自定義的認證程式。藉此將資料庫所儲存的使用者帳號、密碼與權限資料，整合到認證流程中。最後介紹加密密碼的方式，以提高安全性。


-----


## 一、串接資料庫
### （一）準備 MongoDB 服務
本文使用 MongoDB 作為資料庫，來儲存使用者的帳號、密碼與權限資料。

讀者可利用 Docker 啟動 MongoDB 的服務，預設的 port 號是 27017。
``` sh
docker run -d --name "MongoDB_4.4.29" -p 27017:27017 mongo:4.4.29
```

回到程式專案，請在 pom.xml 檔案添加 MongoDB 的依賴。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

接著在「application.properties」配置檔中，添加連線字串。以下是連線到叫做「school」的資料庫。
``` properties
spring.data.mongodb.uri=mongodb://localhost:27017/school
```

最後可以試著啟動 Spring Boot。若 console 沒有出現 exception 訊息，代表有成功連線到 MongoDB 的服務。

### （二）使用者 Model
為了將使用者資料儲存到資料庫，以下設計一個叫做「Member」的類別。其包含使用者的 id、帳號、密碼與權限，共 4 個欄位。
``` java
@Document(collection = "member")
public class Member {

    @Id
    private String id;
    private String username;
    private String password;
    private List<MemberAuthority> authorities = new ArrayList<>();

    // getter, setter ...
}
```

而一個使用者可能包含多個權限，本文設計有「學生」、「老師」與「管理員」這 3 個權限。
``` java
public enum MemberAuthority {
    STUDENT, TEACHER, ADMIN
}
```

上述的 Member 類別使用了 `@Document` 注解，讓該類別的物件在資料庫中，會被儲存到叫做「member」的集合（collection），概念上相當於資料表（table）。

至於 `@Id` 注解，則代表該欄位會對應到 MongoDB 為每一筆資料自動產生的唯一編號（類似主鍵），在此直接做為使用者 id。

### （三）Repository 層
透過 Spring Data 框架，我們可以在程式碼中輕鬆存取資料庫。

以下建立一個叫做「MemberRepository」的介面。它繼承了 `MongoRepository` 介面，並傳入「Member」到泛型類別，代表與資料庫中的對應 collection 做串接。
``` java
@Repository
public interface MemberRepository extends MongoRepository<Member, String> {
    Member findByUsername(String username);
}
```

在 `MongoRepository` 介面中，已經內含 `findById`、`deleteById` 等多個方法。此處我們自行宣告了叫做「findByUsername」的方法，代表要查詢 username 欄位相等的資料。

Spring Data 會在啟動 Spring Boot 時，自動解讀方法名稱，背地產生能實現這些目的的程式。更多介紹可參考筆者的「<a href="/articles/spring-boot-data-mongodb-repository-crud/" target="_blank">【Spring Boot】第8.2課－使用 Spring Data 存取 MongoDB 資料庫，進行基本 CRUD 操作</a>」文章。

## 二、準備 RESTful API
本節會在 Controller 建立多支 RESTful API，目的是為了在本文第六節測試不同權限的帳號，能否被 Spring Security 授權存取。如下表：
| API | 用途 | 授權對象 |
|-|-|-|
|POST /members|建立使用者，並回傳 id|所有人，無論是否登入|
|GET /members|取得所有使用者|管理員|
|GET /selected-courses|回傳「修課清單」字串|學生|
|GET /course-feedback|回傳「課程回饋」字串|老師|
|GET /home|回傳「系統首頁」字串|已登入的任何人|

以下為 Controller 的程式碼，部份 API 會呼叫本文第一節建立的 MemberRepository 來存取 MongoDB 資料庫。
``` java
@RestController
public class MyController {

    @Autowired
    private MemberRepository memberRepository;

    @PostMapping("/members")
    public String createMember(@RequestBody Member member) {
        member.setId(null);
        memberRepository.insert(member);
        return member.getId();
    }

    @GetMapping("/members")
    public List<Member> getMembers() {
        return memberRepository.findAll();
    }

    @GetMapping("/selected-courses")
    public String selectedCourses() {
        return "修課清單";
    }

    @GetMapping("/course-feedback")
    public String courseFeedback() {
        return "課程回饋";
    }

    @GetMapping("/home")
    public String home() {
        return "系統首頁";
    }
}
```

## 三、配置 Spring Security
### （一）一般配置
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
                        .requestMatchers(HttpMethod.POST, "/members").permitAll()
                        .requestMatchers(HttpMethod.GET, "/members").hasAuthority("ADMIN")
                        .requestMatchers(HttpMethod.GET, "/selected-courses").hasAuthority("STUDENT")
                        .requestMatchers(HttpMethod.GET, "/course-feedback").hasAnyAuthority("TEACHER", "ADMIN")
                        .anyRequest().authenticated()
                )
                .formLogin(Customizer.withDefaults())
                .csrf(csrf -> csrf.disable())
                .build();
    }
}
```

呼叫 `authorizeHttpRequests` 方法，可進一步設計各個 API 的授權規則。這裡依照本文第二節的說明進行實作，在此不贅述。

呼叫 `formLogin` 方法，可啟用內建的登入畫面，便於我們在瀏覽器做測試。

### （二）停用 CSRF 保護
呼叫 `csrf` 方法，可進一步停用對於「跨站請求偽造」（Cross-Site Request Forgery， CSRF）攻擊的保護機制。

所謂的 CSRF 攻擊，是指惡意網站利用瀏覽器存取外部網域的 API 時，會自動攜帶目標網域 Cookie 的特性，偷偷存取我們先前使用過的正常網站。

上一篇我們在登入畫面通過認證後，Spring Security 其實會儲存含有認證資訊的 Cookie 在瀏覽器中。之後存取其他 GET API 時，瀏覽器都會自動攜帶此 Cookie。

然而建立使用者的 `POST /members` 這支 API，是無法用瀏覽器存取，必須透過 Postman 或前端程式才行。雖然該 API 被設為公開，但簡單來說，由於 Postman 發送請求時缺少 Cookie，因此會遭到 Spring Security 的 CSRF 保護機制阻擋。

本系列文章中，為了能順利使用 Postman 存取 API，一律停用此機制。而實務開發中，若系統本來就不使用 Cookie，可以直接停用也無妨。但若有使用 Cookie，則需考慮進行另外的配置來防禦 CSRF 攻擊。

## 四、UserDetailsService 認證服務
### （一）UserDetails 介面
在上一篇，我們透過 `InMemoryUserDetailsManager` 元件，建立 in-memory 的測試使用者。示意程式如下：
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
        return new InMemoryUserDetailsManager(List.of(user1));
    }
}
```

而本文的目的是在資料庫儲存使用者。在這之前，讓我們先認識 Spring Security 認證的相關介面，才能了解如何進行抽換。

首先根據上面的示意程式，`InMemoryUserDetailsManager` 的建構子接收了 `UserDetails` 物件。`UserDetails` 是一個介面，在 Spring Security 中都是透過該介面來傳遞使用者資料。

該介面提供了 7 個方法，如下：
``` java
public interface UserDetails extends Serializable {
    String getUsername();
    String getPassword();
    Collection<? extends GrantedAuthority> getAuthorities();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

其中 `getUsername`、`getPassword` 與 `getAuthorities` 是最重要的 3 個方法，分別用來取得帳號、密碼與權限。關於權限的部份，是透過 `GrantedAuthority` 介面的物件來傳遞，到了本文第五節會有相關實作。

至於另外 4 個方法，從名稱便能看出是回傳使用者的各種「狀態」，包含帳號過期、帳號鎖定、密碼過期、是否啟用等。登入時，即便帳號與密碼正確，我們也能設計成請 Spring Security 不要讓具有異常狀態的使用者通過認證。

Spring Security 內建了一個實作 `UserDetails` 介面的類別，那就是 `User`。在建立物件時，我們也能賦予這些狀態，只要其中 1 個是 false（預設值為 true），則將導致認證失敗。

### （二）UserDetailsService 介面
若觀看 `InMemoryUserDetailsManager` 類別的原始碼，會發現它頂層實作了 `UserDetailsService` 介面。

該介面是 Spring Security 用來進行認證的重要元件。它提供一個叫做 `loadUserByUsername` 的方法，用途是接收帳號的值，並回傳內含使用者資料的 `UserDetails` 介面物件。

繼續追蹤 `InMemoryUserDetailsManager` 的原始碼，會發現它是用 Map 資料結構來儲存使用者。
``` java
public class InMemoryUserDetailsManager implements UserDetailsManager {
    // 用 Map 儲存使用者
    private final Map<String, MutableUserDetails> users = new HashMap();

    // ...

    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 從 Map 取得使用者
        UserDetails user = (UserDetails) this.users.get(username.toLowerCase());
        
        if (user == null) {
            throw new UsernameNotFoundException(username);
        } else {
            return new User(
                user.getUsername(),
                user.getPassword(),
                user.isEnabled(),
                user.isAccountNonExpired(),
                user.isCredentialsNonExpired(),
                user.isAccountNonLocked(),
                user.getAuthorities()
            );
        }
    }
}
```

在上一篇的練習中，Spring Security 是將我們在登入畫面輸入的帳號，傳入 `UserDetailsService.loadUserByUsername` 方法。等到該方法將使用者資料包裝成 `UserDetails` 物件後回傳，其 `getPassword` 方法又會被呼叫，並與登入畫面的密碼進行比對。當帳號與密碼相符，則認證成功。

下圖以 UML 的類別圖，呈現這些介面與類別的關係。
<img src="{{ permalink }}spring-security-user-details-service-class-diagram.jpg" />

從圖中下方，讀者可察覺只要提供作自定義的 `UserDetailsService`，就能調整認證方式。

## 五、實作自定義認證服務
啟動程式時，Spring Security 會檢查專案中是否有 `UserDetailsService` 元件。若無，則自動建立一個 `InMemoryUserDetailsManager` 元件，並附帶一個使用者，帳號為「user」，密碼為隨機（可在 console 找到）。

若程式專案中存在我們自行提供的 `UserDetailsService` 元件，例如上一篇手動建立了 `InMemoryUserDetailsManager`，則 Spring Security 會自動採用。

同樣的道理，本文為了抽換成資料庫中的使用者資料，我們需要實作自己的 `UserDetailsService`。
``` java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private MemberRepository memberRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Member member = memberRepository.findByUsername(username);
        if (member == null) {
            throw new UsernameNotFoundException("Can't find member: " + username);
        }

        List<SimpleGrantedAuthority> authorities = member.getAuthorities()
                .stream()
                .map(auth -> new SimpleGrantedAuthority(auth.name()))
                .toList();

        return User
                .withUsername(username)
                .password(member.getPassword())
                .authorities(authorities)
                .build();
    }
}
```

範例程式中實作了 `loadUserByUsername` 方法，它的 `username` 參數，來自於登入畫面所輸入的帳號。

接著呼叫本文第一節實作的 MemberRepository 來查詢資料庫，得到我們自己設計的使用者資料。當找不到使用者，就拋出例外，代表認證失敗。最後將帳號、密碼與權限包裝成 `UserDetails` 物件，回傳給 Spring Security。

權限資料是透過 `GrantedAuthority` 介面來傳遞。Spring Security 內建了一個叫做 `SimpleGrantedAuthority` 的實作類別，此處在建構子中傳入權限的名稱，如本文第一節所設計的學生、老師與管理員。

以生活情境來比喻撰寫這段程式的過程，就像是求職時，即便我們準備了自製的履歷（Member 類別），但依然要填寫公司內部的制式履歷（`UserDetails` 介面），因公司只認制式的人事資料。

## 六、密碼加密
### （一）PasswordEncoder 元件
在使用資料庫儲存密碼時，實務上會先將密碼的原文加密後才儲存，稱之為「密文」。用意是為了保護客戶的資料，避免資料庫內容外洩，或者員工監守自盜。

本節的目標就是對密碼加密。讓我們先認識 Spring Security 的 `PasswordEncoder` 元件，請讀者建立該介面的元件。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
}
```

Spring Security 內建數個 `PasswordEncoder` 的實作類別。此處暫時選擇 `NoOpPasswordEncoder`，代表不加密。對應到上一篇建立的 `InMemoryUserDetailsManager` 元件，就相當於在密碼的值加上 `{noop}` 的前綴。

該介面提供 2 個重要的方法，如下：
``` java
public interface PasswordEncoder {
    String encode(CharSequence rawPassword);
    boolean matches(CharSequence rawPassword, String encodedPassword);
}
```

其中 `encode` 方法的用途，是將密碼原文透過某種演算法運算後，得到密文。而 `matches` 方法，則是將密碼原文進行運算後，與另外傳入的密文做比對，藉此確認密碼是否相符。

在登入畫面進行帳密認證時，Spring Security 會呼叫 `PasswordEncoder.matches` 方法，將我們輸入的密碼原文，加密成密文。接著再比對該值是否與 `UserDetailsService` 回傳 `UserDetails` 物件所包含的密碼密文相同。

讓我們在 Controller 注入 `PasswordEncoder`，並在建立使用者時對密碼做加密。由於目前的實作類別是 `NoOpPasswordEncoder`，因此密文就等於原文。
``` java
@RestController
public class MyController {

    @Autowired
    private MemberRepository memberRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @PostMapping("/members")
    public void createMember(@RequestBody Member member) {
        // 加密密碼
        String encodedPwd = passwordEncoder.encode(member.getPassword());
        member.setPassword(encodedPwd);
        
        member.setId(null);
        memberRepository.insert(member);
    }

    // ...
}
```

### （二）建立測試使用者
請透過 Postman 之類的工具，存取 `POST /members` 這個 API，分別建立 3 個測試使用者。Request body 如下：
``` json
{
    "username": "user1",
    "password": "111",
    "authorities": ["STUDENT"]
}

{
    "username": "user2",
    "password": "222",
    "authorities": ["TEACHER"]
}

{
    "username": "user3",
    "password": "333",
    "authorities": ["TEACHER", "ADMIN"]
}
```

第一個使用者為學生身份；第二個為老師身份；第三個是兼具老師與管理員身份。

此時讀者可分別登入這些帳號，在瀏覽器前往 `/selected-courses`、`/course-feedback`、`/members` 與 `/home` 這 4 支 API，確認有符合在本文第二節所設計的授權規則。

### （三）使用 BCrypt 來加密密碼
接下來讓我們抽換成另一種叫做「BCrypt」的加密方式。如此一來，新建立的使用者，其密碼就真的會被加密為看不懂的密文了。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public PasswordEncoder passwordEncoder() {
         return new BCryptPasswordEncoder();
    }
}
```

BCrypt 是相當流行的加密方式。它使用雜湊函數對資料進行運算，且運算後的結果無法被反推回原始資料。此外還透過隨機加鹽（salt）的方式，讓相同的原始資料，每次都能產生不同的結果，提升加密的安全性。

此時在登入畫面進行認證，由於我們所輸入密碼的原文也會被加密，因此資料庫中原本儲存的未加密密碼將隨之失效，畢竟比對方式的細節已經改變。

下圖是再建立一位有管理員權限的使用者後（帳號為 user4，密碼為 444），在瀏覽器查看所有使用者資料的示意畫面。可看見密碼已經是密文了。
<img src="{{ permalink }}spring-security-member-list.png" />

到目前為止，我們都是透過 Spring Security 的登入畫面進行認證，藉此在瀏覽器存取 Controller 中的 GET API。但該畫面只是方便在學習時進行測試，一旦前後端分離，終究會獨立於 Spring Boot 之外，用各種 HTTP 方法對 API 發出 request。

下一篇會介紹「HTTP Basic」這項認證方式，經由在 request header 攜帶帳密，來存取受保護的 API。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch12.2-security-authentication-integrating-with-mongodb-database)。

上一課：<a href="/articles/spring-boot-security-authentication-and-authorization/" target="_blank">【Spring Boot】第12.1課－初探 Spring Security 的認證與授權</a>

下一課：<a href="/articles/spring-boot-security-http-basic-authentication/" target="_blank">【Spring Boot】第12.3課－在 Spring Security 使用 HTTP Basic 認證</a>