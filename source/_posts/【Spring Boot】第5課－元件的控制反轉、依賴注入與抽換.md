---
title: 【Spring Boot】第5課－元件的控制反轉、依賴注入與抽換
date: 2019-06-04 15:27:40
permalink: articles/spring-boot-bean-ioc-di-and-swap
index_img: /img/index_img/spring-boot.jpg
excerpt: 控制反轉（IoC）與依賴注入（DI）是 Spring Boot 中的重要觀念。本文首先讓讀者知道物件的依賴關係，接著說明「單例」的重要性。隨後經由這些議題來介紹控制反轉與依賴注入，以及應用的方式。最後帶入物件導向的「多型」特性，示範以介面來操作物件，將有助於實作細節的抽換。
categories:
  - ["Spring Boot"]
---

控制反轉（IoC）與依賴注入（DI）是 Spring Boot 中的重要觀念。而筆者選擇在<a href="/articles/spring-boot-three-tier-architecture" target="_blank">第 4 課（三層式架構）</a>結束，練習用專案的架構成形後，才開始介紹。

本文首先透過範例專案，讓讀者知道裡頭那些用來封裝程式邏輯的物件，其實存在著依賴關係。接著說明在後端程式運行期間，為何要求這些物件只能存在唯一一個，以及如何做到。

有了先備的概念後，筆者再經由這些議題，開始介紹控制反轉與依賴注入。最後引進物件導向的「多型」特性，示範以介面來操作物件，有助於實作細節的抽換。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch04-three-tier-architecture)。

## 一、依賴關係
在介紹控制反轉與依賴注入的概念前，讓我們先看看範例專案大概的架構。

以下是產品和使用者的 Repository 層，它們使用 Java 的 Map 資料結構代替真實 DB。並且提供一些方法供外部呼叫，藉此對資料做存取。
``` java
public class ProductRepository {
    private static final Map<String, ProductPO> productMap = new HashMap<>();

    public ProductPO getOneById(String id) {
        return productMap.get(id);
    }

    // ...
}
```
``` java
public class UserRepository {
    private static final Map<String, UserPO> userMap = new HashMap<>();

    public UserPO getOneById(String id) {
        return userMap.get(id);
    }

    // ...
}
```

以下是產品的 Service 層，提供了商業邏輯。該類別擁有全域的 Repository 物件，讓商業邏輯的程式碼呼叫。
``` java
public class ProductService {
    private static final ProductRepository productRepository = new ProductRepository();
    private static final UserRepository userRepository = new UserRepository();

    public ProductVO getProductVO(String id) {
        var productPO = productRepository.getOneById(id);
        // ...

        var userPO = userRepository.getOneById(productPO.getCreatorId());
        // ...

        return productVO;
    }

    // ...
}
```

以下是 Controller 層，提供了 RESTful API。該類別擁有全域的 Service 物件。
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {
    private static final ProductService productService = new ProductService();
    // ...
}
```

讀者可以看出它們之間的關係：Controller 呼叫 Service；而 Service 呼叫 Repository。這種「誰呼叫誰」的關係，正式的稱呼為「依賴」（depend）。

附帶一提，範例專案為了簡便，並未實作使用者的 Controller 和 Service。不然原則上也會是「UserController」依賴「UserService」；而 UserService 依賴 UserRepository 的關係。
``` java
@RestController
public class UserController {
    private static final UserService userService = new UserService();
    // ...
}
```
``` java
public class UserService {
    private static final UserRepository userRepository = new UserRepository();
    // ...
}
```

## 二、元件與單例的概念
本節將討論，同樣是封裝程式邏輯，為何要特別建立物件呢？並且向讀者介紹「單例」（singleton）的概念。

### （一）元件
無論是 ProductService、ProductRepository 或 UserRepository，雖然它們都被建立成物件，但目的都是封裝程式邏輯，而非攜帶資料到處傳遞。

像這種用途的物件，我們給它一個更正式的稱呼，叫做「元件」，英文為「bean」或「component」。元件可以提供方法，來實現商業邏輯、資料處理或存取 DB 等功能。

既然只是用來封裝邏輯，那為什麼不宣告成靜態（static）的方法就好呢？這樣連物件都不必建立了。示意如下：
``` java
public class ProductService {
    private ProductService() {}

    // 宣告為靜態方法
    public static ProductVO getProductVO(String id) {
        // ...
    }
}
```
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductVO> getProduct(@PathVariable("id") String productId) {
        // 呼叫靜態方法
        var product = ProductService.getProductVO(productId);

        // ...
    }
}
```
要知道，系統功能是可以很複雜的，若元件一律提供靜態方法，那就不能善用物件導向中，繼承與多型的特性了。

### （二）單例
筆者先前建立元件的全域變數，其用意是避免每呼叫一次方法，就建立一次元件，示意如下：
``` java
public class ProductService {

    public ProductVO getProductVO(String id) {
        var productPO = new ProductRepository().getOneById(id);
        // ...

        var userPO = new UserRepository().getOneById(productPO.getCreatorId());
        // ...
    }

    // ...
}
```

建立物件會在記憶體佔一個空間，而用完就又要回收，其實沒必要如此反覆。更別提在用戶多的系統，伺服器是很忙碌的。

為了確保應用程式運行期間，特定類別的物件只會存在一個，並能讓各個地方共享，於是出現了「單例」的概念。「單」是單一的意思，而「例」是實例（instance）。

讀者在網路上，能找到各種單例的程式寫法，以下是簡單的範例：
``` java
public class ProductRepository {
    private static final ProductRepository INSTANCE  = new ProductRepository();

    private ProductRepository() {
        // 初始化...
    }

    // 提供取得唯一物件的方法
    public static ProductRepository getInstance() {
        return INSTANCE;
    }
}
```

但也能顧及執行緒安全而變得複雜：
``` java
public class ProductRepository {
    private static ProductRepository INSTANCE;

    private ProductRepository() {
        // 初始化...
    }

    // 提供取得唯一物件的方法
    public static ProductRepository getInstance() {
        if (INSTANCE == null) {
            synchronized (ProductRepository.class) {
                if (INSTANCE == null) {
                    INSTANCE = new ProductRepository();
                }
            }
        }
        return INSTANCE;
    }
}
```

元件的依賴關係，可能會無意間建立出多餘的物件。而手動實現單例，又很不方便。幸好 Spring Boot 有提供「控制反轉」和「依賴注入」的功能，來解決這些問題。

## 三、控制反轉（Inversion of Control，IoC）
在 Java 語言中建立物件的方式，是在想要的地方使用 new 關鍵字。而「控制反轉」的精神，則是將建立物件的工作轉移給外界。

也就是說，無論是透過建構子或 setter 方法，只要物件並非在類別內部建立，而是由外部提供，那就可以稱之為控制反轉。

那要如何讓 Spring Boot 建立單例物件呢？做法是在類別冠上特定的注解（annotation）。Spring Boot 啟動時，會透過 Java 的「反射」（reflection）機制，掃描專案中的哪些類別具有這些注解。

可使用的注解如下，它們有不同的涵義。
* `@Controller`：代表提供 Web API 的表現層。`@RestController` 注解便是繼承於它。於<a href="/articles/spring-boot-implement-restful-api-in-controller" target="_blank">第 3.1 課</a>介紹。
* `@Service`：代表商業邏輯層。
* `@Repository`：代表資料存取層。
* `@Configuration`：代表這裡搭配 `@Value` 注解，存放了「application.properties」配置檔的值。於<a href="/articles/spring-boot-application-properties-configuration" target="_blank">第 6 課</a>介紹。或者搭配 `@Bean` 注解，控制元件的初始化過程。於<a href="/articles/spring-boot-construct-bean-programmatically" target="_blank">第 7 課</a>介紹。
* `@Component`：以上 4 項皆繼承自此注解，是泛用的選擇。

接著，我們在範例專案中的元件類別，冠上適當的注解。
``` java
@Service
public class ProductService {
    // ...
}
```
``` java
@Repository
public class ProductRepository {
    // ...
}
```
``` java
@Repository
public class UserRepository {
    // ...
}
```

如此一來，Spring Boot 建立出單例物件後，便會放在記憶體中管理。這個地方稱為「IoC 容器」。為了方便說明，筆者後續都用「元件」來指稱 IoC 容器中的單例物件。

## 四、依賴注入（Dependency Injection，DI）
### （一）介紹
第一節展示的範例專案，說明了元件的依賴關係。而依賴注入便是要取代原先在程式碼中 new 物件的做法。

類別必須具備前面提到的注解，才能被建立為元件。而 Spring Boot 可以把這些元件「注入」到其他的元件。

範例程式中的 ProductService 依賴 ProductRepository 與 UserRepository。因此這些 Repository 元件，就會被注入到 ProductService 元件中。而 ProductController 又依賴 ProductService，故 Controller 也會被注入。

我們可以使用 `@Autowired` 注解，讓 Spring Boot 注入元件。根據使用此注解的方式，又分為「欄位注入」與「建構子注入」。

### （二）欄位注入（Field Injection）
欄位注入的做法，是對元件的全域變數冠上 `@Autowired` 注解。
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {

    @Autowired
    private ProductService productService;
    
    // ...
}
```
``` java
@Service
public class ProductService {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private UserRepository userRepository;

    // ...
}
```

Spring Boot 會先建立出所有的元件，接著才把每個元件所依賴的其他元件逐一注入。因此元件可能有短暫時間處於初始化不完整的空窗期，這是一項缺點。而優點是寫法方便。

### （三）建構子注入（Constructor Injection）
建構子注入的做法，是宣告包含所有依賴的建構子，再對其加上 `@Autowired` 注解。
``` java
@Service
public class ProductService {
    private final ProductRepository productRepository;
    private final UserRepository userRepository;

    @Autowired
    public ProductService(ProductRepository productRepository, UserRepository userRepository) {
        this.productRepository = productRepository;
        this.userRepository = userRepository;
    }
    // ...
}
```

Spring Boot 在建立一個元件時，會先確認它所依賴的其他元件是否都建立好了。是的話，便從建構子注入進來。否則就先建立其他元件。意即建立與注入元件是同時進行的。

這項做法的缺點是讓程式碼變得較冗長。然而瑕不掩瑜，優點是確保元件在被使用時，已經處於完整初始化的狀態。另外也有利於撰寫單元測試（unit test），因為我們可以將設計好的「模擬物件」（mock）由建構子傳入。

Spring 官方也建議採取這樣的注入方式。

## 五、使用介面注入元件
為了善用物件導向的「多型」特性，我們可以讓元件的類別實作「介面」。而進行依賴注入時，則以介面代替類別。這樣有助於未來更換元件時，不會影響到外部使用的方式。

### （一）多型呼叫
請讀者透過 Java 語言回想一下，當我們讓類別實作「介面」（interface）時，必須完成介面所定義的 public 方法。

當建立這種類別的物件時，宣告的型態可以用介面來取代類別。比方說「ArrayList」與「LinkedList」這兩種資料結構，均實作「List」介面。

用 List 宣告後，使用物件一律都是呼叫該介面所定義的方法。而不必在意實際上是 ArrayList 或者 LinkedList 在運作。示意如下：
``` java
public static void main(String[] args) {
    List<String> arrayList = new ArrayList<>();
    List<String> linkedList = new LinkedList<>();
    process(arrayList);
    process(linkedList);
}

private static void process(List<String> list) {
    list.add("A");
    list.add("B");
    list.add("C");

    for (var i = 0; i < list.size(); i++) {
        System.out.println(list.get(i));
    }

    list.clear();
}
```

回到範例專案，ProductRepository 使用 Map 資料結構來儲存資料。我們同樣也能為 Repository 層設計一個介面，並開發出其他儲存方式的元件。例如用 List 儲存，甚至連接到真實的 DB。

而 ProductService 則固定以該介面操作 Repository 元件。

### （二）實作介面注入
以下設計一個叫做「IProductRepository」的介面，並進行實作。此處為避免混淆，故將原先的 ProductRepository 改名為「MapProductRepository」，強調儲存方式。
``` java
public interface IProductRepository {
    ProductPO getOneById(String id);
    ProductPO insert(ProductPO product);
    void update(ProductPO product);
    void deleteById(String id);
    List<ProductPO> getMany(ProductRequestParameter param);
}
```
``` java
@Repository
public class MapProductRepository implements IProductRepository {
    private static final Map<String, ProductPO> productMap = new HashMap<>();
    // ...

    public ProductPO getOneById(String id) {
        return productMap.get(id);
    }

    public void deleteById(String id) {
        productMap.remove(id);
    }

    // ...
}
```

接著調整 ProductService，改以 IProductRepository 介面來注入。
``` java
@Service
public class ProductService {
    private final IProductRepository productRepository;
    private final UserRepository userRepository;

    public ProductService(IProductRepository productRepository, UserRepository userRepository) {
        this.productRepository = productRepository;
        this.userRepository = userRepository;
    }

    // ...
}
```

經過這樣的修改，ProductService 呼叫的都是 IProductRepository 介面的方法，而實際執行的是被注入元件的內部邏輯。

現在 Spring Boot 啟動時，就會尋找專案中有哪些元件實作了 IProductRepository 介面。目前只有一個，理所當然注入 MapProductRepository。

## 六、抽換相同介面的元件
為了說明如何在相同介面更換元件，筆者準備了另一個實作 IProductRepository 介面的元件。下面是以 List 來儲存產品資料，程式碼寫法將完全不同。
``` java
@Repository
public class ListProductRepository implements IProductRepository {
    private static final List<ProductPO> productList = new ArrayList<>();
    // ...
    
    public ProductPO getOneById(String id) {
        return productList
                .stream()
                .filter(p -> p.getId().equals(id))
                .findFirst()
                .orElse(null);
    }

    public void deleteById(String id) {
        productList.removeIf(p -> p.getId().equals(id));
    }

    // ...
}
```

上面的 ListProductRepository 需依照介面的規範，完成所有 public 方法。完整的範例程式，請參考文末附上的專案。

此時專案中存在多個相同介面的元件。由於 Spring Boot 不知道要注入哪一個，所以啟動會失敗，錯誤訊息節錄如下：
``` text
Parameter 0 of constructor in com.example.demo.service.ProductService required a single bean, but 2 were found:
    - listProductRepository: ...
    - mapProductRepository: ...
...
Consider marking one of the beans as @Primary, updating the consumer to accept multiple beans, or using @Qualifier to identify the bean that should be consumed
```

為了告訴 Spring Boot 要注入哪一個元件，我們可在其中一個類別冠上 `@Primary` 注解。
``` java
@Repository
@Primary
public class ListProductRepository implements IProductRepository {
    // ...
}
```

這會讓所有依賴 IProductRepository 介面的地方，都注入 ListProductRepository。

或者也能透過 `@Qualifier` 注解，在不同的元件注入不同的依賴。
``` java
@Service
public class ProductService {
    private final IProductRepository productRepository;
    private final UserRepository userRepository;

    @Autowired
    public ProductService(@Qualifier("mapProductRepository") IProductRepository productRepository,
                          UserRepository userRepository) {
        this.productRepository = productRepository;
        this.userRepository = userRepository;
    }

    // ...
}
```

以上是在建構子注入的場合使用 `@Qualifier` 注解，需傳入「元件名稱」作為參數。

元件名稱預設是類別名稱的駝峰字。若想自定義，可在 `@Repository`、`@Service` 等元件的注解，傳入參數做設定。
``` java
@Repository("mapProductRepository")
public class MapProductRepository implements IProductRepository {
    // ...
}
```

至於在欄位注入的場合使用 `@Qualifier` 注解，則與 `@Autowired` 注解一併冠在全域變數之上即可。
``` java
@Service
public class ProductService {

    @Autowired
    @Qualifier("mapProductRepository")
    private final IProductRepository productRepository;

    // ...
}
```

本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch05-bean-ioc-di-and-swap)。