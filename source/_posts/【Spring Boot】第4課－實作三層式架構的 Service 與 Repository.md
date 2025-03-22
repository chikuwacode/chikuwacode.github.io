---
title: 【Spring Boot】第4課－實作三層式架構的 Service 與 Repository
date: 2019-06-02 16:04:37
permalink: articles/spring-boot-three-tier-architecture
index_img: /img/index_img/spring-boot.jpg
excerpt: 在前面的課程，我們在 Controller 撰寫了一些資料處理的程式碼。但若全部都寫在 Controller，整個類別就會很龐大。本課會先介紹分層架構的概念，再帶著讀者將資料處理的程式碼，拆分成三個層次。
categories:
  - ["Spring Boot"]
---

在<a href="/articles/spring-boot-implement-restful-api-in-controller/" target="_blank">第 3.1 課</a>與<a href="/articles/spring-boot-use-query-string-and-header-in-controller/" target="_blank">第 3.2 課</a>，我們只有 Controller 這個地方，能撰寫程式來處理請求。然而若全部都寫在這裡，整個類別就會很龐大。

本文將介紹「三層式架構」，並搭配範例專案，點出當前做法所隱含的問題。接著進行修改，進一步將程式碼片段抽離到 Service 與 Repository 兩個層次，讓讀者體會切分的過程與思路。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch03.2-use-query-string-and-header-in-controller)。

## 一、範例專案概觀
在正式介紹三層式架構前，先讓我們稍微認識目前的練習用專案。

以下是 Controller 的其中兩支 API，用途是建立和取得產品資料。本文並未串接真實的 DB，故以 Java 的 Map 資料結構代替。其中 key 為產品 id，而 value 為整筆資料。
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {
    private static final Map<String, Product> productMap = new HashMap<>();
    // ...

    @PostMapping
    public ResponseEntity<Void> createProduct(@RequestBody Product product) {
        // 驗證資料
        if (product.getId() == null) {
            return ResponseEntity.badRequest().build();
        }

        // 檢查合理性
        if (productMap.containsKey(product.getId())) {
            return ResponseEntity.unprocessableEntity().build();
        }

        // 儲存
        productMap.put(product.getId(), product);

        // 組裝 response
        var uri = ServletUriComponentsBuilder
                .fromCurrentRequestUri()
                .path("/{id}")
                .build(Map.of("id", product.getId()));
        return ResponseEntity.created(uri).build();
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable("id") String productId) {
        var product = productMap.get(productId);
        return product == null
                ? ResponseEntity.notFound().build()
                : ResponseEntity.ok(product);
    }
}
```

以下是目前用來攜帶產品資料的類別，包含編號、名稱與價格，共 3 個欄位。
``` java
public class Product {
    private String id;
    private String name;
    private int price;
    
    // getter, setter ...
}
```

## 二、三層式架構的目的與做法
### （一）介紹
在<a href="/articles/spring-boot-implement-restful-api-in-controller/" target="_blank">第 3.1 課</a>，筆者提到 MVC 架構是為了將不同用途的程式分門別類，不要通通寫在一起。在前後端分離的系統中，V（view）屬於前端的工作。

三層式架構的目標，是將屬於後端的 M（Model）與 C（Controller）做進一步的切分。那麼要如何進行呢？這會包含「分層」與「設計資料規格」兩項工作。

### （二）分層
請讀者回頭看看第一節的範例程式，createProduct 這支 API 處理方法，混合了不同目的的程式碼片段，包含：
1. 檢查 request body 是否有缺漏的資料。
2. 檢查產品 id 是否已存在。
3. 儲存產品資料。
4. 建立出 response 的內容。

由於這段程式並沒有很長，所以看起來還好。倘若隨著日後開發，程式碼越來越多，則未來可能會形成不易維護的現象，畢竟要閱讀一大片程式。

實務上會根據各個程式碼片段的目的，歸類為以下三層，也就是在程式中拆成三種類別。
* 表現層：負責接收請求，接著呼叫其他程式進行處理，最後給予回應。其實指的就是 Controller，它是與前端最靠近的層次。
* 資料存取層：負責與 DB 進行 CRUD 的操作。類別名稱常以「Repository」或「Dao」（Data Access Object）結尾。
* 商業邏輯層：負責進行資料處理，類別名稱常以「Service」結尾。商業邏輯是為了滿足情境而開發出的程式邏輯，比方說「觀看產品時要知道賣家資料」、「產品不存在時要出現提示畫面」、「有人留言時要收到通知」等，這都是情境。

這三個層次會透過方法呼叫來互動。原則上是 Controller 呼叫 Service，而 Service 再呼叫 Repository。

分層也未必只能分為三層，當系統太複雜，視情況再繼續細分也沒關係。以下列出幾項分層的好處：
* 幫助我們在開發新功能，或修復問題時，判斷該從何處下手。例如沒收到 query string，要去確認 Controller 層。
* 隔離實作細節，限制修改程式的範圍。例如 Repository 層的儲存方式想從範例中的 Map 換成 MySQL，則只要在這一層修改即可，其他類別理論上就不必調整。
* 相同目的的程式碼，能被重複利用。比方說一支「取得多個產品」的程式，它就可以被「顯示產品列表」和「顯示訂單明細」這兩項功能利用。這也意味著，修改一處，所有呼叫它的地方都會生效。

### （三）設計資料規格
再請讀者看看第一節的範例程式。Product 類別除了用於 request 與 response 的 body，同時也是儲存在 Map 的格式。

假設我們還想在 DB 紀錄產品的建立時間，於是添加叫做「createdTime」的欄位，打算由後端寫入，如下：
``` java
public class Product {
    private String id;
    private String name;
    private int price;
    private long createdTime;

    // getter, setter ...
}
```

但不知道的人單獨看這個類別，搞不好會以為我們在建立與更新的 request body 中，開放前端攜帶這項時間資料呢！

再假設前端想顯示產品建立者的名字，於是我們添加「creatorName」欄位。並且經過資料庫正規化後，DB 還需紀錄使用者 id 來做關聯，欄位叫做「creatorId」，如下：
``` java
public class Product {
    private String id;
    private String name;
    private int price;
    private long createdTime;
    private String creatorId;
    private String creatorName;

    // getter, setter ...
}
```

同樣的道理，別人可能感到疑惑：為何 DB 的產品資料會同時儲存使用者的 id 與名字呢，不是有做正規化嗎？

從這兩個例子可察覺，當一個類別身兼多個用途，可能造成誤解。不該把 request 和 response body 的欄位，直接套用到 DB。

架構中的三個層次，在呼叫方法時，伴隨資料的輸入與輸出。因此，除了分層之外還有一項工作，那就是設計層次間溝通的資料規格。也就是說，攜帶產品資料的類別，會依照用途而建立出多個。比方說 request 專用、response 專用、DB 專用等。

## 三、實作不同的資料規格
在實作 Service 與 Repository 層之前，本節先準備好各種攜帶產品資料的類別，作為層次之間溝通的規格。

以下的 ProductCreateRequest 類別，會做為建立資料時的 request body。包含產品 id、名稱、價格與建立者 id，共 4 個欄位。
``` java
public class ProductCreateRequest {
    private String id;
    private String name;
    private int price;
    private String creatorId;

    // getter, setter ...
}
```

以下的 ProductUpdateRequest 類別，會做為更新資料時的 request body。但只允許更新名稱與價格這 2 個欄位。
``` java
public class ProductUpdateRequest {
    private String name;
    private int price;

    // getter, setter ...
}
```

以下的 ProductPO 類別，是對應到 DB 儲存的欄位。「PO」的全稱為「Persistence Object」，有持久儲存的意思。它比上述兩個 request 類別，多了建立與更新時間這 2 個欄位。
``` java
public class ProductPO {
    private String id;
    private String name;
    private int price;
    private String creatorId;
    private long createdTime;
    private long updatedTime;

    public static ProductPO of(String id, String name, int price, String creatorId) {
        var po = new ProductPO();
        po.id = id;
        po.name = name;
        po.price = price;
        po.creatorId = creatorId;
        po.createdTime = Instant.now().getEpochSecond();
        po.updatedTime = p.createdTime;

        return po;
    }

    // getter, setter ...
}
```

以下的 ProductVO 類別，是用來攜帶要回傳給前端的資料。「VO」的全稱為「View Object」，有展示的意思。它比 ProductPO 多了建立者名稱這個欄位。
``` java
public class ProductVO {
    private String id;
    private String name;
    private int price;
    private String creatorId;
    private String creatorName;
    private long createdTime;
    private long updatedTime;
    
    // getter, setter ...
}
```

本文想讓產品能透過 creatorId 欄位，關聯到使用者資料。因此再額外建立使用者的類別。
``` java
public class UserPO {
    private String id;
    private String name;

    public static UserPO of(String id, String name) {
        var u = new UserPO();
        u.id = id;
        u.name = name;
        
        return u;
    }

    // getter, setter ...
}
```

完成後，就能開始將原先寫在 Controller 的程式碼抽離到其他層次。過程中，也會運用這些類別來攜帶資料。

## 四、實作 Repository 層
### （一）宣告類別與測試資料
以下是產品和使用者的 Repository 類別。裡頭內建了測試資料，使用 Map 資料結構模擬出一個 DB。
``` java
public class ProductRepository {
    private static final Map<String, ProductPO> productMap = new HashMap<>();

    static {
        Stream.of(
                ProductPO.of("P1", "Android Development (Java)", 380, "U1"),
                ProductPO.of("P2", "Android Development (Kotlin)", 420, "U2")
        ).forEach(p -> productMap.put(p.getId(), p));
    }
    // TODO
}
```
``` java
public class UserRepository {
    private static final Map<String, UserPO> userMap = new HashMap<>();

    static {
        Stream.of(
                UserPO.of("U1", "Vincent"),
                UserPO.of("U2", "Ivy")
        ).forEach(u -> userMap.put(u.getId(), u));
    }
    // TODO
}
```

接下來要提供 CRUD 的方法，讓其他地方能藉此存取 DB。

### （二）取得一筆資料
以下是在產品和使用者的 Repository，實作「透過 id 取得一筆資料」的方法。
``` java
public class ProductRepository {
    private static final Map<String, ProductPO> productMap = new HashMap<>();
    // ...

    public ProductPO getOneById(String id) {
        return productMap.get(id);
    }
}
```
``` java
public class UserRepository {
    private static final Map<String, UserPO> userMap = new HashMap<>();
    // ...

    public UserPO getOneById(String id) {
        return userMap.get(id);
    }
}
```

此處很單純地從 Map 取出資料。而回傳的型態是代表 DB 欄位的 ProductPO 與 UserPO。

### （三）新增資料
以下是實作「新增產品資料」的方法。
``` java
public class ProductRepository {
    private static final Map<String, ProductPO> productMap = new HashMap<>();
    // ...

    public ProductPO insert(ProductPO product) {
        if (product.getId() == null) {
            product.setId(generateRandomId());
        }

        if (productMap.containsKey(product.getId())) {
            throw new RuntimeException();
        }

        product.setCreatedTime(Instant.now().getEpochSecond());
        product.setUpdatedTime(product.getCreatedTime());
        productMap.put(product.getId(), product);

        return product;
    }
    
    private String generateRandomId() {
        var uuid = UUID.randomUUID().toString();
        return uuid.substring(0, uuid.indexOf("-"));
    }
}
```

此處設計成產品 id 可由外部自行提供，或者讓內部產生隨機字串。接著 Repository 層會寫入資料的建立時間，隨後儲存。

若 id 發生重複，則透過拋出例外（exception）的方式，停止處理這項請求。這裡暫時使用 `RuntimeException`，我們留到第五節再來調整。

### （四）更新與刪除資料
以下是更新與刪除產品資料的方法。
``` java
public class ProductRepository {
    private static final Map<String, ProductPO> productMap = new HashMap<>();
    // ...

    public void update(ProductPO product) {
        if (!productMap.containsKey(product.getId())) {
            throw new RuntimeException();
        }

        product.setUpdatedTime(Instant.now().getEpochSecond());
        productMap.put(product.getId(), product);
    }

    public void deleteById(String id) {
        productMap.remove(id);
    }
}
```

更新時，我們將傳入 Repository 的 ProductPO 物件，視為已經攜帶著新的資料。在實作上，會先透過 id 確認該資料是否存在。是的話，就寫入更新時間後儲存，否則拋出 exception 來中止。

刪除時，則直接從 Map 移除，不做檢查。站在 Repository 層最靠近 DB 的立場，它只要確保 Map 不要有該 id 的資料即可。

### （五）取得多筆資料
基本上就是將原先寫在 Controller 的 getProducts 方法，幾乎原封不動地搬過來。只是要將回傳值型態改為代表 DB 欄位的 ProductPO。
``` java
public class ProductRepository {
    // ...

    public List<ProductPO> getMany(ProductRequestParameter param) {
        // ...
    }
}
```

實作內容在此不贅述，讀者可參考文末附上的完成後專案。

## 五、用 Exception 回傳 HTTP 狀態碼
在練習用專案中，我們在 Controller 是透過 ResponseEntity 物件回傳 HTTP 狀態碼，如 404（Not Found）、422（Unprocessable Entity）等，讓前端知道有問題發生。

但等到將程式碼抽離到其他層次後，就不適合這麼做了，畢竟 `ResponseEntity` 是被設計用來回應給前端，不建議出現在擔任其他職責的層次。

本節介紹如何在拋出 exception 中止程式流程時，能回傳 HTTP 狀態碼。

以 404 和 422 的狀態碼為例，我們可以像這樣建立例外類別：
``` java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class NotFoundException extends RuntimeException {
    public NotFoundException(String msg) {
        super(msg);
    }
}
```
``` java
@ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
public class UnprocessableEntityException extends RuntimeException {
    public UnprocessableEntityException(String msg) {
        super(msg);
    }
}
```

它們繼承了 RuntimeException，並冠上 `@ResponseStatus` 注解來設定狀態碼。

接著請回到 ProductRepository，改為拋出這種 exception。
``` java
public class ProductRepository {
    private static final Map<String, ProductPO> productMap = new HashMap<>();
    // ...

    public ProductPO insert(ProductPO product) {
        // ...

        if (productMap.containsKey(product.getId())) {
            throw new UnprocessableEntityException("Product id " + product.getId() + " is existing.");
        }

        // ...
    }

    public void update(ProductPO product) {
        if (!productMap.containsKey(product.getId())) {
            throw new UnprocessableEntityException("Product " + product.getId() + " doesn't exist.");
        }

        // ...
    }
}
```

附帶一提，若讀者除了 HTTP 狀態碼，還想想透過 response body 提供前端更詳細的資訊，請參考「[【Spring Boot】使用 Controller Advice 處理例外與 query string](https://chikuwa-tech-study.blogspot.com/2023/02/spring-boot-controller-advice-handle-exception-and-query-string.html)」文章。

## 六、實作 Service 層
### （一）宣告類別
完成資料存取層後，本節接續進行商業邏輯層的實作。
``` java
public class ProductService {
    // Should use @Autowired in the future
    private static final ProductRepository productRepository = new ProductRepository();
    private static final UserRepository userRepository = new UserRepository();

    // TODO
}
```

上面宣告了一個 Service 類別，準備實作產品的商業邏輯。

此處還用 `new` 建立了 ProductRepository 和 UserRepository 的全域變數，以便呼叫前面第五節封裝好的程式。

如果讀者以前已經碰過一點 Spring Boot，看到這邊應該覺得奇怪：為何不使用 `@Autowired` 注解，而是用 new 的？

原因是筆者想在<a href="/articles/spring-boot-bean-ioc-di-and-polymorphism/" target="_blank">第 5 課</a>單獨用一篇文章，來介紹「控制反轉」及「依賴注入」這兩項關於 Spring 的重要觀念，故本文暫時用 new 的方式。

### （二）取得資料
由於代表 DB 欄位的類別為 ProductPO，若想將其變成前端需要的 ProductVO 規格，那就得先準備轉換的方法。以下就只是將兩個類別的欄位一一對應罷了。
``` java
public class ProductVO {
    // ...

    public static ProductVO of(ProductPO po) {
        var vo = new ProductVO();
        vo.id = po.getId();
        vo.name = po.getName();
        vo.price = po.getPrice();
        vo.creatorId = po.getCreatorId();
        vo.createdTime = po.getCreatedTime();
        vo.updatedTime = po.getUpdatedTime();

        return vo;
    }
    // ...
}
```

至於 ProductVO 的 creatorName 欄位的資料，可在 Service 層進行取得的動作。
``` java
public class ProductService {
    private static final ProductRepository productRepository = new ProductRepository();
    private static final UserRepository userRepository = new UserRepository();

    public ProductVO getProductVO(String id) {
        var productPO = getProductPO(id);
        var productVO = ProductVO.of(productPO);
        
        var user = userRepository.getOneById(productPO.getCreatorId());
        productVO.setCreatorName(user.getName());
        
        return productVO;
    }

    private ProductPO getProductPO(String id) {
        var po = productRepository.getOneById(id);
        if (po == null) {
            throw new NotFoundException("Product " + id + " doesn't exist.");
        }

        return po;
    }
}
```

首先從 Repository 層取得產品資料，將其轉換為 ProductVO。接著取得使用者的名字後，再填入進去。

當然，如果沒有該 id 的產品，就回傳 404 狀態碼。

### （三）建立資料
由於後續會由第三節設計的 ProductCreateRequest 類別接收前端的請求，因此也要準備一個轉換成 ProductPO 的方法。
``` java
public class ProductPO {
    // ...

    public static ProductPO of(ProductCreateRequest req) {
        var po = new ProductPO();
        po.id = req.getId();
        po.name = req.getName();
        po.price = req.getPrice();
        po.creatorId = req.getCreatorId();

        return po;
    }
    // ...
}
```

而 Service 的實作內容如下：
``` java
public class ProductService {
    private static final ProductRepository productRepository = new ProductRepository();
    private static final UserRepository userRepository = new UserRepository();
    // ...

    public ProductPO createProduct(ProductCreateRequest productReq) {
        var userPO = userRepository.getOneById(productReq.getCreatorId());
        if (userPO == null) {
            throw new UnprocessableEntityException("Product creator " + productReq.getCreatorId() + " doesn't exist.");
        }

        var productPO = ProductPO.of(productReq);
        productPO = productRepository.insert(productPO);

        return productPO;
    }
}
```

首先檢查是否有這位建立者。有的話，便將 ProductCreateRequest 轉換為 ProductPO，再呼叫 Repository 儲存即可。

### （四）更新與刪除資料
以下是更新與刪除產品資料的邏輯。
``` java
public class ProductService {
    private static final ProductRepository productRepository = new ProductRepository();
    // ...

    public void updateProduct(String id, ProductUpdateRequest productReq) {
        var productPO = getProductPO(id);
        productPO.setName(productReq.getName());
        productPO.setPrice(productReq.getPrice());
        productRepository.update(productPO);
    }

    public void deleteProduct(String id) {
        var productPO = getProductPO(id);
        productRepository.deleteById(productPO.getId());
    }

    private ProductPO getProductPO(String id) {
        var po = productRepository.getOneById(id);
        if (po == null) {
            throw new NotFoundException("Product " + id + " doesn't exist.");
        }

        return po;
    }
}
```

更新時，會從 Repository 層取得資料，並將 ProductUpdateRequest 物件的欄位值更新上去，最後再儲存。

刪除時，會先嘗試透過 id 取得資料。若不存在，則回傳 404 狀態。否則呼叫 Repository 層進行刪除。

### （五）取得多筆資料
基本上就是直接呼叫 Repository 的方法，取得多個 ProductPO。接著再仿照前面取得一筆產品資料的做法，將建立者的名字寫入，組成完整的 ProductVO。
``` java
public class ProductService {
    // ...

    public List<ProductVO> getProductVOs(ProductRequestParameter param) {
        var productPOs = productRepository.getMany(param);

        // ...
    }
}
```

要留意的是，實務上應盡量避免在迴圈中存取 DB，以免效能太差。具體實作細節，讀者可參考文末附上的完成後專案。

## 七、調整 Controller 層
第五節實作了 Repository，而第六節讓 Service 的商業邏輯呼叫 Repository。最後我們要讓 Controller 只呼叫 Service，讓它專注於自己的職責。
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {
    // Should use @Autowired in the future
    private static final ProductService productService = new ProductService();

    @GetMapping("/{id}")
    public ResponseEntity<ProductVO> getProduct(@PathVariable("id") String productId) {
        var product = productService.getProductVO(productId);
        return ResponseEntity.ok(product);
    }

    @PostMapping
    public ResponseEntity<Void> createProduct(@RequestBody ProductCreateRequest request) {
        var product = productService.createProduct(request);
        // ...

        return ResponseEntity.created(uri).build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<Void> updateProduct(
            @PathVariable("id") String productId, @RequestBody ProductUpdateRequest request) {
        productService.updateProduct(productId, request);
        return ResponseEntity.noContent().build();
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable("id") String productId) {
        productService.deleteProduct(productId);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<ProductVO>> getProducts(
            @ModelAttribute ProductRequestParameter param
    ) {
        var products = productService.getProductVOs(param);
        return ResponseEntity.ok(products);
    }
}
```

這裡同樣暫時用 new 的方式，建立 ProductService 物件。並且以 ProductCreateResquest 和 ProductUpdateRequest 類別接收請求，而給予回應則使用 ProductVO 類別。

如此一來，Controller 的每一支 API 處理方法，不再有過長的程式碼，凸顯它只有接收請求、呼叫處理、給予回應這些工作，變得簡潔許多！


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch04-three-tier-architecture)。

上一課：<a href="/articles/spring-boot-use-query-string-and-header-in-controller/" target="_blank">【Spring Boot】第3.2課－在 Controller 接收 query string 與操作 header</a>

下一課：<a href="/articles/spring-boot-bean-ioc-di-and-swap/" target="_blank">【Spring Boot】第5課－元件的控制反轉、依賴注入與抽換</a>