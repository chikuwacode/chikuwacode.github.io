---
title: 【Spring Boot】第12.4課－從 Security Context 取得 API 存取方的認證資訊
date: 2023-11-03 13:28:00
permalink: articles/spring-boot-security-context-authentication-info
index_img: /img/index_img/spring-boot.jpg
excerpt: 在開發商業邏輯時，我們經常需要知道 API 存取方的身份是誰。本文將以 HTTP Basic 認證方式為例，示範如何從「Security Context」取得 API 存取方的使用者資訊。最後透過觀察原始碼，認識 Spring Security 進行 HTTP Basic 認證的原理。
categories:
  - ["Spring Boot"]
  - ["Spring Security"]
---

在開發商業邏輯時，我們經常需要知道 API 存取方的身份。舉例來說，在網路發表文章、按讚和留言，理論上都要在資料庫中紀錄「誰發的文」、「誰按的讚」和「誰留的言」，亦即資料的建立者。

本文將以 HTTP Basic 認證方式為例，示範如何從「Security Context」取得 API 存取方的使用者資料。接著準備自定義的使用者類別，以便在程式邏輯中能有更多的資料可運用。

最後透過觀察原始碼，認識 Spring Security 進行 HTTP Basic 認證的原理，讓讀者知道該認證資訊從何而來。


-----


## 一、配置 Spring Security
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
                .httpBasic(Customizer.withDefaults())
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(requests -> requests.anyRequest().permitAll())
                .build();
    }
}
```

呼叫 `httpBasic` 方法，可啟用「HTTP Basic」的認證方式。這樣在發送 request 時，就能在「Authorization」這個 header 攜帶帳密，讓 Spring Security 知道我們的身份。詳情可參考<a href="/articles/spring-boot-security-http-basic-authentication/" target="_blank">第 12.3 課</a>。

呼叫 `csrf` 方法，可進一步停用對於 CSRF 攻擊的保護機制，讓 Postman 工具能順利存取 API。

呼叫 `authorizeHttpRequests` 方法，可進一步定義各個 RESTful API 的授權規則。此處設定為 API 不需通過認證也可存取。

### （二）準備使用者
以下準備了 1 個測試使用者，帳號為「user1」，密碼為「111」，權限為「學生」與「助理」。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public UserDetailsService inMemoryUserDetailsManager() {
        UserDetails userDetails = User
                .withUsername("user1")
                .password("{noop}111")
                .authorities("STUDENT", "ASSISTANT")
                .build();
        return new InMemoryUserDetailsManager(List.of(userDetails));
    }
}
```

附帶一提，`InMemoryUserDetailsManager` 本身就有實作 `UserDetailsService` 介面。若讀者想改為串接真實的資料庫，只要基於此介面進行抽換即可，做法可參考<a href="/articles/spring-boot-security-authentication-integrating-with-mongodb-database/" target="_blank">第 12.2 課</a>。

## 二、取得 Security Context 的認證資訊
以下是在 Controller 中準備一支 RESTful API，用途是存取時，能得到自身的使用者資料。
``` java
@RestController
public class MyController {

    @GetMapping("/home")
    public String home() {
        SecurityContext context = SecurityContextHolder.getContext();
        Authentication auth = context.getAuthentication();
        Object principal = auth.getPrincipal();

        if ("anonymousUser".equals(principal)) {
            return "你尚未經過身份認證";
        }
        
        UserDetails userDetails = (UserDetails) principal;
        return String.format(
                "你好，%s，你的權限是：%s",
                userDetails.getUsername(),
                userDetails.getAuthorities()
        );
    }
}
```

在 API 的程式邏輯中，第一行呼叫了 `SecurityContextHolder` 類別的靜態方法，取得了 `SecurityContext` 物件。第二行從 `SecurityContext` 物件取得 `Authentication` 物件。

呼叫 `Authentication.getPrincipal` 方法，可得到 API 存取方的使用者資料。但該值會隨著認證情況的不同而有差異，所以回傳值型態是 `Object`。

若尚未通過認證，principal 的值固定會是「anonymousUser」字串。若通過認證，則為 `UserDetails` 物件。在範例程式中，我們取出帳號與權限的值進行運用。

下圖是使用 Postman，以 HTTP Basic 認證方式在 request header 攜帶帳密後存取 API。
<img src="{{ permalink }}spring-security-context-postman-get-simple-info.png" />

而下圖是以未認證的身份，匿名存取 API。
<img src="{{ permalink }}spring-security-context-postman-get-anonymous-user.png" />

上面見到了許多類別與介面，到了本文第五節，筆者會介紹相關的原始碼，讓讀者了解原理。

## 三、自定義使用者資料
### （一）設計使用者類別
上一節的範例程式，取用了使用者的帳號與權限這兩項資料。然而在實務上，「使用者」這種資料通常還會有更多欄位，例如 id、名字或信箱等。

接下來讓我們準備自定義的使用者類別，取名為「Member」。它除了基本的帳號、密碼與權限，還包含了 id、暱稱與信箱，共 6 個欄位。
``` java
public class Member {
    private String id;
    private String username;
    private String password;
    private String nickname;
    private String email;
    private List<MemberAuthority> authorities = new ArrayList<>();
    
    // getter, setter ...
}
```
``` java
public enum MemberAuthority {
    STUDENT, ASSISTANT
}
```

在 Spring Security 中，會以 `UserDetails` 介面來傳遞使用者資料。若想攜帶自定義的資料，我們也需準備一個實作該介面的類別。
``` java
public class MemberUserDetails implements UserDetails {
    private final Member member;

    public MemberUserDetails(Member member) {
        this.member = member;
    }

    public Member getMember() {
        return member;
    }

    // 回傳自定義內容
    public String getId() {
        return member.getId();
    }

    public String getNickname() {
        return member.getNickname();
    }

    public String getEmail() {
        return member.getEmail();
    }

    // 實作介面規範的方法
    public String getUsername() {
        return member.getUsername();
    }

    public String getPassword() {
        return member.getPassword();
    }

    public Collection<? extends GrantedAuthority> getAuthorities() {
        return member.getAuthorities()
                .stream()
                .map(Enum::name)
                .map(SimpleGrantedAuthority::new)
                .toList();
    }

    public boolean isEnabled() {
        return true;
    }

    public boolean isAccountNonExpired() {
        return true;
    }

    public boolean isAccountNonLocked() {
        return true;
    }

    public boolean isCredentialsNonExpired() {
        return true;
    }
}
```

這個「MemberUserDetails」類別實作了 `UserDetails` 介面，並完成介面所規範的方法。此外還提供其他自定義的方法，以取得使用者的 id、暱稱與信箱。這些資料都源自於從建構子傳入的 Member 物件。

### （二）實作認證程式
本文第一節的 `InMemoryUserDetailsManager` 所「回傳」的 `UserDetails`，它的實作類別固定是內建的 `User`，我們無法使它回傳先前設計好的 MemberUserDetails。

因此請讀者準備另一個實作 `UserDetailsService` 介面的類別來代替，讓它在 Spring Security 進行認證時被使用。
``` java
public class UserDetailsServiceImpl implements UserDetailsService {
    // 模擬在資料庫儲存使用者資料
    private final Map<String, Member> memberMap = new HashMap<>();

    public UserDetailsServiceImpl(List<Member> members) {
        members.forEach(member -> memberMap.put(member.getUsername(), member));
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Member member = memberMap.get(username);
        if (member == null) {
            throw new UsernameNotFoundException("Can't find username: " + username);
        }

        return new MemberUserDetails(member);
    }
}
```

接著在 Spring Security 的配置類別中，將它建立為元件。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public UserDetailsService userDetailsService() {
        Member member = new Member();
        member.setId("1");
        member.setUsername("user1");
        member.setPassword("111");
        member.setNickname("One");
        member.setEmail("user1@gmail.com");
        member.setAuthorities(List.of(MemberAuthority.STUDENT, MemberAuthority.ASSISTANT));

        return new UserDetailsServiceImpl(List.of(member));
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
}
```

此處同時將自定義的 Member 物件傳入，模擬出在資料庫中儲存使用者。如此就能把 `UserDetails` 介面的實作類別，換成自定義的 MemberUserDetails 了。

另外也需建立 `PasswordEncoder` 元件，以指定密碼的加密方式，筆者選擇不加密。

## 四、封裝取得認證資訊的邏輯
在第二節的 Controller 範例程式中，我們呼叫 `SecurityContextHolder` 的方法取得 `SecurityContext`，接著再取得 `Authentication`，最後取得認證資訊 principal。

由於太繁瑣了，因此本節的目的是將這段過程封裝成一個元件。筆者取名為「UserIdentity」，代表用途是取得身份。
``` java
@Component
public class UserIdentity {
    private static final MemberUserDetails ANONYMOUS_USER = new MemberUserDetails(new Member());

    private MemberUserDetails getMemberUserDetails() {
        SecurityContext context = SecurityContextHolder.getContext();
        Authentication auth = context.getAuthentication();
        Object principal = auth.getPrincipal();

        return "anonymousUser".equals(principal)
                ? ANONYMOUS_USER
                : (MemberUserDetails) principal;
    }

    public boolean isAnonymous() {
        return getMemberUserDetails() == ANONYMOUS_USER;
    }

    public String getId() {
        return getMemberUserDetails().getId();
    }

    public String getUsername() {
        return getMemberUserDetails().getUsername();
    }

    public String getNickname() {
        return getMemberUserDetails().getNickname();
    }

    public String getEmail() {
        return getMemberUserDetails().getEmail();
    }

    public List<MemberAuthority> getAuthorities() {
        return getMemberUserDetails().getMember().getAuthorities();
    }
}
```

類別中宣告了「getMemberUserDetails」方法，將取得認證資訊 principal 的過程封裝起來。若認證成功，便轉型為 MemberUserDetails，供其他方法呼叫。

若為匿名存取 API，則回傳空值的物件。此處運用了「Null Object Pattern」設計模式，避免其他方法取用 MemberUserDetails 物件時，意外導致「Null Pointer Exception」例外。

此元件提供了取得 id、帳號、信箱、權限以及是否匿名等使用者資料的方法。讓我們將其注入到 Controller 中，在程式邏輯取得這些資料做運用。
``` java
@RestController
public class MyController {

    @Autowired
    private UserIdentity userIdentity;

    @GetMapping("/home")
    public String home() {
        if (userIdentity.isAnonymous()) {
            return "你尚未經過身份認證";
        }

        return String.format(
                "嗨，你的編號是%s%n帳號：%s%n暱稱：%s%n信箱：%s%n權限：%s",
                userIdentity.getId(),
                userIdentity.getUsername(),
                userIdentity.getNickname(),
                userIdentity.getEmail(),
                userIdentity.getAuthorities()
        );
    }
}
```

下圖是以「user1」的使用者身份存取 API。
<img src="{{ permalink }}spring-security-context-postman-get-customized-info.png" />

如此一來，讀者在開發商業邏輯時，不論是要在資料庫儲存文章的建立者，還是將暱稱寫入訊息文字，又或者是寄送郵件，都能更輕鬆地取得當前 API 存取方的使用者資料。

## 五、HTTP Basic 認證的原理
### （一）Filter 原始碼
我們已經知道如何取用 `SecurityContext` 中的認證資訊，而本節將介紹該內容是從何而來。

Spring Security 是透過 Java Servlet 的「Filter」，才能在 request 到達 Controller 前進行認證與授權。其中 `BasicAuthenticationFilter` 是專門處理 HTTP Basic 認證的 Filter。

以下是它的部份原始碼，供讀者參考。
``` java
public class BasicAuthenticationFilter extends OncePerRequestFilter {
    private AuthenticationConverter authenticationConverter = new BasicAuthenticationConverter();
    private AuthenticationManager authenticationManager;
    private SecurityContextHolderStrategy securityContextHolderStrategy = SecurityContextHolder.getContextHolderStrategy();
    // ...
    
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        try {
            // 把 HTTP request 轉換成等待認證的資料
            Authentication authRequest = this.authenticationConverter.convert(request);
            if (authRequest == null) {
                // ...
            }

            String username = authRequest.getName();
            if (this.authenticationIsRequired(username)) {
                // 底層呼叫 UserDetailsService 與 PasswordEncoder 進行認證
                Authentication authResult = this.authenticationManager.authenticate(authRequest);
                
                // 將認證成功的資訊放入 SecurityContext
                SecurityContext context = this.securityContextHolderStrategy.createEmptyContext();
                context.setAuthentication(authResult);
                this.securityContextHolderStrategy.setContext(context);
                
                // ...
            }
        } catch (AuthenticationException var8) {
            // ...
        }

        chain.doFilter(request, response);
    }
    
    protected boolean authenticationIsRequired(String username) {
        // ...
    }
}
```

本節會提到該 Filter 中的 3 個重要元件：`AuthenticationConverter`、`AuthenticationManager` 與 `SecurityContextHolderStrategy`。接下來筆者將搭配 UML 的類別圖，分別進行說明。

## （二）AuthenticationConverter 解析帳密
HTTP Basic 是一種經由在 request header 攜帶帳號與密碼，讓後端進行身份認證的方式。

Filter 接收到 request 後，首先 `AuthenticationConverter` 會從 `HttpServletRequest` 取出「Authorization」這個 header 的值，進行 Base64 解碼。解析出帳密後，封裝成 `Authentication` 物件。
<img src="{{ permalink }}spring-boot-security-authentication-converter-class-diagram.jpg" />

而 `AuthenticationConverter` 介面的實作類別是 `BasicAuthenticationConverter`，它會回傳 `Authentication` 介面的實作類別 `UsernamePasswordAuthenticationToken`。

解析出的帳號，會放在 `UsernamePasswordAuthenticationToken` 物件的 `principal` 欄位，而密碼放在 `credentials` 欄位。

### （三）AuthenticationManager 進行認證
將帳號與密碼的值封裝成 `Authentication` 介面的物件後，接下來 `AuthenticationManager` 會進行身份認證，並回傳另一個新的 `Authentication` 物件。兩個物件的差別在於，新物件會包含認證後的結果，即使用者資料。

這個 `AuthenticationManager` 介面，有個實作類別叫做 `ProviderManager`，它擁有多個 `AuthenticationProvider` 介面的物件，讀者可理解成「認證功能的提供者」。
<img src="{{ permalink }}spring-boot-security-authentication-manager-class-diagram.jpg" />

在 HTTP Basic 認證的情況下，會由 `DaoAuthenticationProvider` 提供認證功能。它接收 `Authentication` 物件後，使用了 `UserDetailsService` 與 `PasswordEncoder` 進行帳號與密碼的認證。

認證成功後，`DaoAuthenticationProvider` 會將 `UserDetailsService` 回傳的 `UserDetails` 放入 `UsernamePasswordAuthenticationToken` 物件的 `principal` 欄位中，並以 `Authentication` 介面回傳。

### （四）SecurityContextHolderStrategy 管理認證資訊
Spring Security 會將認證後的使用者資料儲存於記憶體，讓我們在程式邏輯中能使用。所以我們在第二節的 Controller 才能在取得 `UserDetails` 物件。

負責管理認證資訊的是 `SecurityContextHolder` 類別中的 `SecurityContextHolderStrategy`。後者是一個介面，代表管理的「策略」，Spring Security 內建數種實作好的策略。

而前者會在啟動程式時選擇其中一種管理策略。除此之外，就只是對外提供存取認證資訊的方法罷了。
<img src="{{ permalink }}spring-boot-security-context-holder-class-diagram.jpg" />

預設的管理策略是 `ThreadLocalSecurityContextHolderStrategy`，它採用了「ThreadLocal」的技術，讓每個執行緒只能取得屬於自己的資料。因此，即便 Spring Boot 同時處理許多 request，也不會遇到執行緒安全的問題，意外地取得他人的認證資訊。

回到 Filter 的邏輯。接下來會產生一個 `SecurityContext` 介面的物件，將認證後的 `Authentication` 物件封裝起來。

最後將 `SecurityContext` 存回 `SecurityContextHolderStrategy`，就完成 HTTP Basic 認證的流程。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch12.4-security-context-authentication-info)。

上一課：<a href="/articles/spring-boot-security-http-basic-authentication/" target="_blank">【Spring Boot】第12.3課－在 Spring Security 使用 HTTP Basic 認證</a>

下一課：<a href="/articles/spring-boot-security-implement-login-api-with-jwt/" target="_blank">【Spring Boot】第12.5課－將 Spring Security 與 JWT 結合，實作登入 API</a>