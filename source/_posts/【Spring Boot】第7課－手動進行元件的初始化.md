---
title: 【Spring Boot】第7課－手動進行元件的初始化
date: 2020-03-03 14:34:49
permalink: articles/spring-boot-construct-bean-programmatically
index_img: /img/index_img/spring-boot.jpg
excerpt: 我們知道在類別冠上 @Service、@Repository 等注解，就能讓 Spring Boot 建立出元件。但假設我們想讓元件透過 List 或 Map 等成員，來儲存自己的資料或狀態，那該如何對它們進行初始化呢？本文將介紹 @Bean 注解，透過程式碼手動進行元件的建立過程。
categories:
  - ["Spring Boot"]
---

在 Spring Boot 的 IOC 容器中，存放著各種元件。元件除了依賴其他元件，它們也可以擁有自己的資料成員。比方說建立 List、Map 或其它物件的全域變數，來儲存自己的資料，或者說狀態。

但這些用來儲存元件狀態的全域變數，可能需要先完成初始化，才能開始使用。本文將介紹 `@Bean` 注解，讓我們透過程式碼手動控制元件的初始化。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch07-start-construct-bean-programmatically)。

## 一、範例程式介紹
在本文的練習用專案，能找到筆者事先準備好的範例程式。

簡單來說，就是讓 controller 去呼叫 repository，存取使用者資料。而 repository 層又提供 2 種實作方式，分別透過 Map 與 List 資料結構來儲存資料。

以下是使用者的類別。
``` java
public class User {
    private String id;
    private String name;

    public static User of(String id, String name) {
        var u = new User();
        u.id = id;
        u.name = name;

        return u;
    }

    // getter, setter ...
}
```

以下是 repository 層的介面，提供新增、取得與刪除的方法。此介面會有 2 個類別分別去實作。
``` java
public interface IUserRepository {
    void insert(User user);
    User findById(String id);
    void deleteById(String id);
}
```

以下是透過 Map 結構來儲存資料的 repository。
``` java
@Repository
public class MapUserRepository implements IUserRepository {
    private static final Map<String, User> userMap = new HashMap<>();

    public void insert(User user) {
        if (userMap.containsKey(user.getId())) {
            throw new RuntimeException("User id " + user.getId() + " is existing.");
        }

        userMap.put(user.getId(), user);
    }

    public User findById(String id) {
        return userMap.get(id);
    }

    public void deleteById(String id) {
        userMap.remove(id);
    }
}
```

以下是透過 List 結構來儲存資料的 repository。
``` java
@Repository
public class ListUserRepository implements IUserRepository {
    private static final List<User> userList = new ArrayList<>();

    public void insert(User user) {
        var isExisting = userList
                .stream()
                .anyMatch(u -> u.getId().equals(user.getId()));
        if (isExisting) {
            throw new RuntimeException("User id " + user.getId() + " is existing.");
        }

        userList.add(user);
    }

    public User findById(String id) {
        return userList
                .stream()
                .filter(u -> u.getId().equals(id))
                .findFirst()
                .orElse(null);
    }

    public void deleteById(String id) {
        userList.removeIf(u -> u.getId().equals(id));
    }
}
```

以下是 controller，它會依賴 IUserRepository，並透過介面提供的方法來操作。此處採用的實作類別為 MapUserRepository。
``` java
@RestController
@RequestMapping(path = "/users")
public class UserController {

    @Autowired
    @Qualifier("mapUserRepository")
    private IUserRepository userRepository;

    @PostMapping
    public ResponseEntity<Void> createUser(@RequestBody User user) {
        try {
            userRepository.insert(user);
            return ResponseEntity.status(HttpStatus.CREATED).build();
        } catch (RuntimeException e) {
            return ResponseEntity.unprocessableEntity().build();
        }
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable("id") String id) {
        var user = userRepository.findById(id);
        return user == null
                ? ResponseEntity.notFound().build()
                : ResponseEntity.ok(user);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable("id") String id) {
        userRepository.deleteById(id);;
        return ResponseEntity.noContent().build();
    }
}
```

## 二、簡易的初始化做法
在第一節的範例程式中，不論是 MapUserRepository 或 ListUserRepository，它們都透過全域變數來儲存資料。假設我們希望程式啟動時，裡面已經有預先準備好一些初始資料了，那可以怎麼做呢？

若讀者有看過筆者 Spring Boot 系列的前幾篇文章，就知道可以在類別中寫一個 `static` 區塊，如下：
``` java
@Repository
public class MapUserRepository implements IUserRepository {
    private static final Map<String, User> userMap = new HashMap<>();

    static {
        var users = List.of(
                User.of("U1", "Vincent"),
                User.of("U2", "Ivy"),
                User.of("U3", "Dora")
        );
        users.forEach(u -> userMap.put(u.getId(), u));
    }

    // ...
}
```

或者我們也可宣告一個方法，並冠上 `@PostConstruct` 注解，如下：
``` java
@Repository
public class MapUserRepository implements IUserRepository {
    private static final Map<String, User> userMap = new HashMap<>();

    @PostConstruct
    private void init() {
        var users = List.of(
                User.of("U1", "Vincent"),
                User.of("U2", "Ivy"),
                User.of("U3", "Dora")
        );
        users.forEach(u -> userMap.put(u.getId(), u));
    }

    // ...
}
```

這兩種做法的差別，在於執行的時間點。`static` 區塊會在程式一啟動時就執行，並且只能使用類別中的 static 資料成員及方法。

而 `@PostConstruct` 方法，則是在物件建立完成後才會執行。它可以使用 static 與 non-static 的資料成員及方法。由於物件（或者說元件）已經建立完成了，所以當然也能使用注入進來的其他元件。

## 三、使用 @Bean 注解建立元件
### （一）認識 @Bean 注解
不論是 `static` 區塊，還是 `@PostConstruct` 所注解的方法，都能幫助我們完成全域變數的初始化。但這些做法，其實也會增加元件類別的程式碼。

本節將示範如何將元件的初始化過程，從自身的類別中分離出去。具體的做法是將元件所需要的資料或物件，先在外部初始化完成，再透過建構子傳入元件中。

Spring Boot 提供了 `@Bean` 注解，它會冠在元件類別中的方法上。其用途是將該方法的回傳值建立為元件，用法示意如下：
``` java
@Configuration
public class RepositoryConfig {

    @Bean
    public IUserRepository mapUserRepository() {
        return new MapUserRepository();
    }
}
```

以上建立了 Configuration 類別。接著宣告一個方法，回傳值型態為 IUserRepository 介面，而實際回傳的是 MapUserRepository 物件。

如此一來，Spring Boot 便會將 MapUserRepository 建立為元件。

### （二）實作初始化過程
既然 `@Bean` 所注解的方法的回傳值會被建立為元件，那在回傳之前，不就可以客製我們想要的初始化過程了嗎？

根據前面筆者提到的做法，接下來就要在該方法中準備好測試資料，再傳入 MapUserRepository 或 ListUserRepository 的建構子中。

首先，請讀者調整這 2 個實作類別，讓它們的建構子能接收使用者資料。同時也要移除 `@Repository` 注解，因為我們即將改為透過 `@Bean` 注解來建立元件。
``` java
public class MapUserRepository implements IUserRepository {
    private final Map<String, User> userMap;

    public MapUserRepository(Map<String, User> userMap) {
        this.userMap = userMap;
    }

    // ...
}
```
``` java
public class ListUserRepository implements IUserRepository {
    private final List<User> userList;

    public ListUserRepository(List<User> userList) {
        this.userList = userList;
    }

    // ...
}
```

接著回到 Configuration 類別，分別宣告方法，對 MapUserRepository 與 ListUserRepository 進行初始化。
``` java
@Configuration
public class RepositoryConfig {

    @Bean(name = "mapUserRepo")
    public IUserRepository mapUserRepository() {
        var users = List.of(
                User.of("U1", "Vincent"),
                User.of("U2", "Ivy"),
                User.of("U3", "Dora")
        );

        var userMap = new HashMap<String, User>();
        users.forEach(u -> userMap.put(u.getId(), u));

        return new MapUserRepository(userMap);
    }

    @Bean(name = "listUserRepo")
    public IUserRepository listUserRepository() {
        var users = new ArrayList<User>();
        users.add(User.of("U1", "Vincent"));
        users.add(User.of("U2", "Ivy"));
        users.add(User.of("U3", "Dora"));

        return new ListUserRepository(users);
    }
}
```

這樣就能把元件初始化的過程，從自身類別中分離出去。

附帶一提，我們可在 `@Bean` 注解中定義元件的名稱（預設為方法名稱），供其他元件指定要注入哪一個實作。因此，controller 仍可透過 `@Qualifier` 注解進行指定。
``` java
public class UserController {

    @Autowired
    @Qualifier("mapUserRepo")
    private IUserRepository userRepository;

    // ...
}
```

## 四、綜合應用
### （一）搭配其他注解
`@Bean` 所注解的元件建立方法，也可搭配其他注解一起使用。

比方說，當有回傳值型態相同的多個方法，可透過 `@Primary` 注解，來指定預設採用的元件。
``` java
@Configuration
public class RepositoryConfig {

    @Primary
    @Bean(name = "mapUserRepo")
    public IUserRepository mapUserRepository() {
        // ...
    }

    @Bean(name = "listUserRepo")
    public IUserRepository listUserRepository() {
        // ...
    }
}
```

另外，我們也可將元件所依賴的其他元件類別或介面，寫在方法參數中。示意如下：
``` java
@Configuration
public class ServiceConfig {
    // ...

    @Bean
    public UserService userService(IUserRepository userRepository) {
        return new UserService(userRepository);
    }
}
```
``` java
public class UserService {
    private IUserRepository userRepository;

    public UserService(IUserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

當然，也能使用 `@Qualifier` 注解，指定要取用的實作類別。
``` java
@Configuration
public class ServiceConfig {
    // ...

    @Bean
    public UserService userService(
        @Qualifier("mapUserRepo") IUserRepository userRepository
    ) {
        return new UserService(userRepository);
    }
}
```

### （二）結合配置檔
在程式中，controller 只會取用其中一個實作類別。而另一個用不到的元件，其實不必建立出來。

因此，我們可將這兩個 `@Bean` 注解的方法合併，再透過 if 判斷決定要建立哪一個。

接下來的範例，筆者會結合<a href="/articles/spring-boot-application-properties-configuration/" target="_blank">第 6 課</a>介紹的 application.properties 配置檔，提供元件初始化的一些選項。請先添加以下 2 個設定值：
``` properties
user-repository.storage=map
user-repository.test-data.amount=3
```

第一個設定值代表要使用何種資料結構來儲存，可選擇「map」或「list」。第二個設定值代表要產生幾筆測試資料。

回到 Configuration 類別，在方法參數中使用 `@Value` 注解，即可將配置檔的設定值讀取進來使用。
``` java
@Configuration
public class RepositoryConfig {

    @Bean
    public IUserRepository userRepository(
            @Value("${user-repository.storage}") String storage,
            @Value("${user-repository.test-data.amount:0}") int amount
    ) {
        // 建立測試資料
        var users = new ArrayList<User>();
        for (var i = 1; i <= amount; i++) {
            var user = User.of("U" + i, "Test User " + i);
            users.add(user);
        }

        // 選擇資料結構
        if ("map".equalsIgnoreCase(storage)) {
            var userMap = new HashMap<String, User>();
            users.forEach(u -> userMap.put(u.getId(), u));
            System.out.println("Create MapProductRepository.");

            return new MapUserRepository(userMap);
        } else if ("list".equalsIgnoreCase(storage)) {
            System.out.println("Create ListProductRepository.");
            return new ListUserRepository(users);
        } else {
            throw new IllegalArgumentException("Please provide correct user repository storage type.");
        }
    }
}
```

在程式邏輯中，首先產生指定數量的測試資料（預設為 0 筆）。接著根據設定值是指定 Map 或 List 資料結構，再建立出對應的 repository 物件。

同時也將測試資料傳入 repository 的建構子中，隨即回傳，完成元件的初始化。

最後讀者別忘了確認注入 IUserRepository 的 Controller，已經不必再透過 `@Qualifier` 注解，來指定要注入的元件了。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch07-fin-construct-bean-programmatically)。


上一課：<a href="/articles/spring-boot-application-properties-configuration/" target="_blank">【Spring Boot】第6課－在 application.properties 配置檔提供設定值（以 Java Mail 為例）</a>

下一課：<a href="/articles/spring-boot-mongodb-introduction-and-setup/" target="_blank">【Spring Boot】第8.1課－MongoDB 介紹與準備資料庫環境</a>