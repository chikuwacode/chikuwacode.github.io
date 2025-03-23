---
title: 【Spring Boot】第12.6課－實作 Spring Security 的認證 Filter（以 JWT 為例）
date: 2021-06-06 15:58:00
permalink: articles/spring-boot-security-implement-authentication-filter-with-jwt
index_img: /img/index_img/spring-boot.jpg
excerpt: JWT 經常被攜帶於 request header 中，用來表明自己的身份。本文將以 JWT 為基礎，實作一個認證 Filter。過程中會將 header 中的 JWT 轉化為認證後的使用者資料。最後在 Controller 中透過 Security Context 取用。
categories:
  - ["Spring Boot"]
  - ["Spring Security"]
---

上一篇實作了建立與解析 JWT 的程式。JWT 經常被攜帶於 request header 中，用來表明自己的身份，與 HTTP Basic 認證需攜帶帳密的 Base64 編碼有異曲同工之妙。

本文將以 JWT 為基礎，實作一個性質與 HTTP Basic 相似的認證 Filter。首先會說明練習用專案大致有哪些程式。接著開發 Filter，將 header 中的 JWT 轉化為認證後的使用者資料。最後在 Controller 中取用。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch12.5-security-implement-login-api-with-jwt)。

## 一、程式專案概觀
本文的範例程式，是基於上一篇的完成後專案繼續實作。筆者僅展示比較重要的部份，而讀者也能下載專案，一邊對照著看。

### （一）使用者資料
以下是自定義的使用者類別，可假想成對應到資料庫的表。它包含 id、帳號、密碼、暱稱與權限，共 5 個欄位。其中權限包含「學生」與「助理」這 2 種。
``` java
public class Member {
    private String id;
    private String username;
    private String password;
    private String nickname;
    private List<MemberAuthority> authorities = new ArrayList<>();

    // getter, setter ...
}
```
``` java
public enum MemberAuthority {
    STUDENT, ASSISTANT
}
```

以下是自定義的 `UserDetails` 類別。它在建構子接收了上述的 Member 物件，將當中的值都複製進來。這些值會在後面存取「Security Context」時被使用。
``` java
public class MemberUserDetails implements UserDetails {
    private String id;
    private String username;
    private String password;
    private String nickname;
    private List<MemberAuthority> memberAuthorities;

    public MemberUserDetails() {
    }

    public MemberUserDetails(Member member) {
        this.id = member.getId();
        this.username = member.getUsername();
        this.password = member.getPassword();
        this.nickname = member.getNickname();
        this.memberAuthorities = member.getAuthorities();
    }

    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.memberAuthorities
                .stream()
                .map(Enum::name)
                .map(SimpleGrantedAuthority::new)
                .toList();
    }

    // getter, setter ...
}
```

以下是自定義的 `UserDetailsService` 類別，它在建構子可接收多個 Member 物件。但本文並未串接真實的資料庫，故以 Java 的 Map 資料結構來儲存。
``` java
public class UserDetailsServiceImpl implements UserDetailsService {
    private final Map<String, Member> memberMap = new HashMap<>();

    public UserDetailsServiceImpl(List<Member> members) {
        members.forEach(m -> memberMap.put(m.getUsername(), m));
    }

    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Member member = memberMap.get(username);
        if (member == null) {
            throw new UsernameNotFoundException("Can't find username: " + username);
        }

        return new MemberUserDetails(member);
    }
}
```

### （二）解析 JWT 的程式
以下是處理 JWT 相關事務的類別，其中用來解析 JWT 的方法名稱為「parseToken」。
``` java
public class JwtService {
    private final SecretKey secretKey;
    private final int validSeconds;
    private final JwtParser jwtParser;

    public JwtService(String secretKeyStr, int validSeconds) {
        // ...
    }

    // ...

    public Claims parseToken(String jwt) throws JwtException {
        return jwtParser.parseSignedClaims(jwt).getPayload();
    }
}
```

該方法所回傳的 `Claims` 介面物件，包含了當初放進 JWT 的各種資料，在本文第二節實作認證 Filter 時會取用它們。

### （三）Spring Security 配置
以下是將 `UserDetailsService` 與 JwtService 建立成元件。
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
        member.setAuthorities(List.of(MemberAuthority.STUDENT, MemberAuthority.ASSISTANT));

        return new UserDetailsServiceImpl(List.of(member));
    }
    
    @Bean
    public JwtService jwtService(
            @Value("${jwt.secret-key}") String secretKeyStr,
            @Value("${jwt.valid-seconds}") int validSeconds
    ) {
        return new JwtService(secretKeyStr, validSeconds);
    }
}
```

此處也建立一筆使用者資料，帳號為「user1」，密碼為「111」。我們會在本文第三節進行測試時，用來呼叫登入 API。

## 二、實作認證 Filter
### （一）前言
當 request 到達後端，Spring Security 背後會有許多 Filter 依序執行不同的工作。多個 Filter 合稱為「過濾鏈」（filter chain），其中也包含了認證與授權的流程。當 request 得到授權，才能進入 Controller。

在<a href="/articles/spring-boot-security-context-authentication-info/" target="_blank">第 12.4 課</a>，筆者介紹了 HTTP Basic 認證流程的原始碼，也就是 `BasicAuthenticationFilter`。讓我們先回顧一下該 Filter 的邏輯。

它獲取了「Authorization」這個 request header 的值，其為帳號與密碼的 Base64 編碼。經由解碼得到帳密的值後，隨即進行身份認證。

認證成功後，會得到內含使用者資料的 `UserDetails` 介面物件。將其放入 Security Context 後，即可供其他程式存取。

而本節也要實作相同目的的 Filter。我們會接收 JWT，進行解析後，將裡頭的使用者資料放入 Security Context，相當於完全取代 HTTP Basic 認證。到了本文第三節，將在 Controller 進行測試。

### （二）建立 Filter
首先請建立一個 Filter 類別。
``` java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private static final List<String> EXCLUDED_PATHS = List.of("/login", "/who-am-i");
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        // TODO
        filterChain.doFilter(request, response);
    }
    
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        return EXCLUDED_PATHS.contains(request.getServletPath());
    }
}
```

該類別繼承自 `OncePerRequestFilter`，確保後端每次收到 request，該 Filter 只會執行一次。否則 Spring Security 執行第一次後，因為該 Filter 剛好是個元件（bean），於是 Spring Boot 又執行第二次。

此處特別覆寫了 `shouldNotFilter` 方法。範例程式的 Controller 中，有 `POST /login` 與 `GET /who-am-i` 這兩支 API。我們不預期在存取登入或測試用途的 API 時，還得經過身份認證。藉由在該方法中判斷 API 路徑，可避免 Filter 處理特定的 request。

### （三）處理收到的 JWT
依照同樣的思路，以下會逐步實作出一個接收 header，並處理 JWT 的 Filter。為了進行解析，需注入前面提到的 JwtService。
``` java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private static final String BEARER_PREFIX = "Bearer ";
    // ...

    @Autowired
    private JwtService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        // 取得 header
        String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);
        if (authHeader == null) {
            filterChain.doFilter(request, response);
        }

        // 解析 JWT
        String jwt = authHeader.substring(BEARER_PREFIX.length());
        Claims claims;
        try {
            claims = jwtService.parseToken(jwt);
        } catch (JwtException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        // TODO

        filterChain.doFilter(request, response);
    }

    // ...
}
```

首先從 Authorization 這個 request header 取出值。由於攜帶 JWT 的格式，是以「Bearer」加一個半形空格做為前綴，因此要進行字串的擷取，才能得到 JWT 的值。將其傳入 JwtService 進行解析後，得到 `Claims` 介面的物件。

如果 JWT 因為過期、格式錯誤等原因而導致解析失敗，則回傳 HTTP 401 的狀態碼，就不再將 request 往 Controller 送去了。

接下來筆者將 `Claims` 物件中所包含的使用者資料，轉換為 MemberUserDetails 類別的規格。好處是透過「getId」、「getNickname」等事先設計好的方法來取值，會讓我們更加直覺。
``` java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private static final String BEARER_PREFIX = "Bearer ";
    // ...

    @Autowired
    private JwtService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        // ...

        // 建立 UserDetails 物件
        MemberUserDetails userDetails = new MemberUserDetails();
        userDetails.setId(claims.getSubject());
        userDetails.setUsername(claims.get("username", String.class));
        userDetails.setNickname(claims.get("nickname", String.class));

        List<MemberAuthority> memberAuthorities = ((List<String>) claims.get("authorities"))
                .stream()
                .map(MemberAuthority::valueOf)
                .toList();
        userDetails.setMemberAuthorities(memberAuthorities);
        
        // TODO

        filterChain.doFilter(request, response);
    }

    // ...
}
```

呼叫 `Claims.get` 方法，傳入欄位名稱與要轉換成的型態，即可得到欄位值。要注意的是，該方法僅支援轉換成簡單的 String、Integer、Date 等型態。

若想轉換成其他型態，例如 List、陣列或自定義的類別等，請參考[文件](https://github.com/jwtk/jjwt?tab=readme-ov-file#parsing-of-custom-claim-types)，實作反序列化（deserialize）的方式。

將 `Claims` 中的資料包裝成 MemberUserDetails 後，最後就是要放入 Security Context 了。放入的方式，是提供 `Authentication` 介面的物件。
``` java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    // ...

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        // ...

        // 放入 Security Context
        Authentication token = new UsernamePasswordAuthenticationToken(
                userDetails,
                null,
                userDetails.getAuthorities()
        );
        SecurityContextHolder.getContext().setAuthentication(token);

        filterChain.doFilter(request, response);
    }

    // ...
}
```

此處選擇的 `Authentication` 為 `UsernamePasswordAuthenticationToken`。它是一個方便的物件，能夠以 Object 型態攜帶使用者資料。也能以 Collection 介面，攜帶權限資料給 Spring Security。

### （四）配置到過濾鏈
實作完 JWT 的認證 Filter 後，我們需要添加到 Spring Security 的 filter chain 中，認證的效果才能生效。
``` java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // ...

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity httpSecurity,
            JwtAuthenticationFilter jwtAuthFilter
    ) throws Exception {
        return httpSecurity
                .csrf(AbstractHttpConfigurer::disable)
                .authorizeHttpRequests(requests -> requests.anyRequest().permitAll())
                .addFilterBefore(jwtAuthFilter, BasicAuthenticationFilter.class)
                .build();
    }
}
```

Spring Security 的 filter chain 會比其他 Filter 還優先執行。而裡頭最後一個執行的 Filter 叫做 `AuthorizationFilter`，正是負責 API 的授權。

因此，若未將自定義的認證 Filter 添加到 Spring Security 中，那麼它就會比 `AuthorizationFilter` 還晚執行。即便程式碼中有提供權限資料給 Security Context，但早就先被判定為不允許授權了。

此處呼叫 `addFilterBefore` 方法，將 JwtAuthenticationFilter 放置在 `BasicAuthenticationFilter` 的前面。選擇相鄰位置的理由，是因為它們的性質相同，筆者認為較不會影響 filter chain 的運作。

## 三、在 Controller 測試
上一節實作出認證的 Filter 後，本節我們會在 Controller 取出 API 存取方的使用者資料進行運用。
``` java
@RestController
public class MyController {
    // ...

    @GetMapping("/home")
    public String home() {
        Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if ("anonymousUser".equals(principal)) {
            return "你尚未經過身份認證";
        }

        MemberUserDetails userDetails = (MemberUserDetails) principal;
        return String.format("嗨，你的編號是%s%n帳號：%s%n暱稱：%s%n權限：%s",
                userDetails.getId(),
                userDetails.getUsername(),
                userDetails.getNickname(),
                userDetails.getAuthorities()
        );
    }
}
```

上面的範例程式中，從 `SecurityContext` 取出了 principal 的資料。如果存取 API 時未攜帶 JWT，principal 的值固定會是「anonymousUser」字串。針對此情形，僅回傳簡單的字串。

若 JwtAuthenticationFilter 有將解析成功的結果轉換為 MemberUserDetails，並放入 `SecurityContext`，則 principal 的值將會是這份 `UserDetails`。

如果讀者對 Security Context 不太了解，可參考<a href="/articles/spring-boot-security-context-authentication-info/" target="_blank">第 12.4 課</a>。或者想將存取 Security Context 的過程封裝成元件，該文亦有範例程式。

下圖是使用 Postman 進行測試的結果。
<img src="{{ permalink }}spring-security-postman-send-jwt-to-get-context.png" />

可看到 API 回傳了 id、帳號、暱稱與權限，成功以 JWT 取代 HTTP Basic 的認證方式。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch12.6-security-implement-authentication-filter-with-jwt)。

上一課：<a href="/articles/spring-boot-security-implement-login-api-with-jwt/" target="_blank">【Spring Boot】第12.5課－將 Spring Security 與 JWT 結合，實作登入 API</a>