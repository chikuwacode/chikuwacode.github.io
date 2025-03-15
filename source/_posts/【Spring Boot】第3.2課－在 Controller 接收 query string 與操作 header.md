---
title: 【Spring Boot】第3.2課－在 Controller 接收 query string 與操作 header
date: 2019-05-26 16:12:50
permalink: articles/spring-boot-use-query-string-and-header-in-controller
index_img: /img/index_img/spring-boot.jpg
excerpt: 本文將繼續介紹 Controller 的其他細節。首先是接收 query string，進行條件篩選與排序。接著是 header 的處理，包含接收指定名稱的 request header，以及產生 URI，在 response header 提供 Location。最後提供幾項實用的技巧來簡化程式碼。
categories:
  - ["Spring Boot"]
---

完成<a href="/articles/spring-boot-implement-restful-api-in-controller" target="_blank">第 3.1 課</a>後，相信讀者已經知道如何在 Controller 設計 API。本文將繼續介紹其他可以實作的細節。

首先是接收查詢字串（query string），進行條件篩選與排序，回傳多筆資料。接著是標頭（header）的處理，包含接收指定名稱的 request header，以及回傳 Location。最後提供幾項實用的做法，讓 Controller 的程式碼更簡潔。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch03.1-implement-restful-api-in-controller)。

## 一、範例專案介紹
首先回顧一下練習用專案中的測試資料和 Controller。

以下的自訂類別，描述了產品的編號、名稱與價格，會作為測試資料。
``` java
public class Product {
    private String id;
    private String name;
    private int price;

    // getter, setter ...
}
```

以下是 Controller，由於沒有串接真實的 DB，因此將測試資料存放在 Java 的 Map 資料結構中。
``` java
@RestController
public class ProductController {
    private static final Map<String, Product> productMap = new HashMap<>();

    static {
        Stream.of(
                Product.of("B1", "Android Development (Java)", 380),
                Product.of("B2", "Android Development (Kotlin)", 420),
                Product.of("B3", "Data Structure (Java)", 250),
                Product.of("B4", "Finance Management", 450),
                Product.of("B5", "Human Resource Management", 330)
        ).forEach(p -> productMap.put(p.getId(), p));
    }

    // TODO
}
```

## 二、接收查詢字串
### （一）範例說明
延續練習用專案，本節讓我們實作新的 API。它能接收端點（endpoint）上的查詢字串（query string），對測試資料 Product 進行搜尋與排序。

筆者設計了三個查詢字串。
* searchKey：搜尋的關鍵字。本文支援搜尋「id」與「name」欄位。
* sortField：用來排序的欄位。本文支援「name」與「price」。
* sortDir：排序的方向。僅支援「asc」（遞增）與「desc」（遞減）。

舉例來說，若想以「manage」關鍵字搜尋，並以價格遞增排序，則 endpoint 寫法為：
`GET /products?searchKey=manage&sortField=price&sortDir=asc`。

### （二）宣告 API
了解需求後，就能開始實作了。以下是在 Controller 先宣告 API 的處理方法。
``` java
@RestController
public class ProductController {
    // ...

    @GetMapping("/products")
    public ResponseEntity<List<Product>> getProducts(
            @RequestParam(value = "searchKey", required = false, defaultValue = "") String keyword,
            @RequestParam(value = "sortField", required = false) String sortField,
            @RequestParam(value = "sortDir", required = false)String sortDir
    ) {
        // TODO
        return null;
    }
}
```

這個方法的三個參數，分別對應到上述的搜尋關鍵字、排序欄位與方向，且都冠上了 `@RequestParam` 注解。

該注解又可再傳入多個參數，用途說明如下。
* `value`：指定要接收 endpoint 上的哪一個 query string。好處是方法參數與 query string 可分開取名，不必使用相同的名稱。否則預設將會與方法參數一樣。
* `required`：設定該 query string 是否為必填。預設為 true，若前端未提供，後端會自動回應 HTTP 400 狀態碼（Bad Request）。
* `defaultValue`：若前端未提供 query string，可在此定義預設值。

最後，由於這支 API 會回傳多筆資料，而且可能有順序性，所以回傳值的 `ResponseEntity`，其泛型類別給予的是 List。

### （三）實作程式邏輯
本文的範例程式並未串接真實的 DB，故透過 Java 的「stream」 API 來進行操作。
``` java
@RestController
public class ProductController {
    private static final Map<String, Product> productMap = new HashMap<>();

    @GetMapping("/products")
    public ResponseEntity<List<Product>> getProducts(
            @RequestParam(value = "searchKey", required = false, defaultValue = "") String keyword,
            @RequestParam(value = "sortField", required = false) String sortField,
            @RequestParam(value = "sortDir", required = false) String sortDir
    ) {
        // 準備排序方式
        Comparator<Product> comparator;
        if ("name".equals(sortField)) {
            comparator = Comparator.comparing(x -> x.getName().toLowerCase());
        } else if ("price".equals(sortField)) {
            comparator = Comparator.comparing(Product::getPrice);
        } else {
            comparator = (a, b) -> 0;
        }

        // 判斷是否遞減
        if ("desc".equalsIgnoreCase(sortDir)) {
            comparator = comparator.reversed();
        }

        // 執行
        var products = productMap.values()
                .stream()
                .filter(x -> x.getName().toLowerCase().contains(keyword.toLowerCase()) ||
                        x.getId().contains(keyword))
                .sorted(comparator)
                .toList();
        return ResponseEntity.status(HttpStatus.OK).body(products);
    }
}
```

此處的程式邏輯分為三個步驟：
1. 根據排序的欄位，宣告 Comparator 物件，用來比較大小。
2. 根據排序的方向，決定是否將 Comparator 調整為遞減。
3. 取得測試資料的集合，呼叫 stream 的相關方法。將搜尋關鍵字用來篩選，再將符合的資料做排序。

完成後，可實際用 Postman 呼叫看看。
<img src="{{ permalink }}postman-get-with-query-string.png" />

本節著重於如何在 Controller 接收 query string。至於檢查它們的值是否合法，不在本文的討論範圍。

## 三、接收與回傳標頭
### （一）回傳 Location 標頭
在<a href="/articles/spring-boot-restful-api" target="_blank">第 2 課</a>有介紹過，如果 API 的功能是建立資源，那麼後端可額外回傳「Location」這個標頭（header），提供指向該資源的 endpoint。

在練習用專案中，尚有另一個 API 是「POST /products」，用途是建立新的產品資料。其對應的 API 處理方法為「createProduct」，本節就在這個 API 提供此 header。
``` java
@RestController
public class ProductController {
    // ...

    @PostMapping("/products")
    public ResponseEntity<Void> createProduct(@RequestBody Product product) {
        // ...

        URI uri = ServletUriComponentsBuilder
                .fromCurrentRequestUri()
                .path("/{id}")
                .build(Map.of("id", product.getId()));
        HttpHeaders headers = new HttpHeaders();
        headers.setLocation(uri);

        return ResponseEntity
                .status(HttpStatus.CREATED)
                .headers(headers)
                .build();
    }
}
```

透過 `ServletUriComponentsBuilder` 提供的方法，我們能一步步建構出 URI 物件。
1. `fromCurrentRequestUri`：從前端呼叫當前 API 的 endpoint 為基準開始建構。例如 `http://localhost:8080/products`。
2. `path`：以目前建構出的結果，繼續往後添加新的路徑，亦可透過「{ }」符號挖空格（placeholder），並於後續填入值。
3. `build`：傳入 Map，將值填入上述的 placeholder 中，並回傳建立好的 URI 物件。

準備好 URI 後，請建立 `HttpHeaders` 物件，添加 Location 的值。`HttpHeaders` 提供一系列的「setXXX」方法，可以很直覺地加上資料。最後將其附帶於 `ResponseEntity` 中。

以下是 Postman 的操作結果，可看見 response header 確實有出現「Location」。
<img src="{{ permalink }}postman-response-header-indicate-location.png" />

### （二）接收標頭
Controller 除了可以回傳 header，當然也能接收。以下的小範例，是接收一個叫做「If-Modified-Since」的 header。
``` java
@GetMapping("/test")
public ResponseEntity<Foo> getFoo(
    @RequestHeader(HttpHeaders.IF_MODIFIED_SINCE) String ifModifiedSince
) {
    // ...
}
```

這裡使用了 `@RequestHeader` 注解，並於注解的參數提供要接收 header 的名稱。

## 四、簡化 Controller 的程式
完成<a href="/articles/spring-boot-implement-restful-api-in-controller" target="_blank">第 3.1 課</a>與第 3.2 課後，讀者在 Controller 將擁有五支 RESTful API。本節將補充一些寫法，讓程式能簡潔好讀一些。

### （一）套用 Endpoint 前綴
這些跟產品有關的 API，在 `@GetMapping`、`@PostMapping` 等注解中，其 endpoint 的開頭都要寫上「/products」，感覺重複性有點高。

我們可以在 Controller 類別冠上 `@RequestMapping` 注解，定義路徑的前綴，之後底下的 endpoint 都會掛上它。
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {

    // GET /products/{id}
    @GetMapping("/{id}")

    // POST /products
    @PostMapping

    // PUT /products/{id}
    @PutMapping("/{id}")

    // DELETE /products/{id}
    @DeleteMapping("/{id}")
}
```

### （二）選擇 HTTP 狀態碼
先前提供 HTTP 狀態碼的方式，是呼叫 `ResponseEntity.status` 方法，傳入 `HttpStatus` 這個列舉（enum）物件。

其實 `ResponseEntity` 類別也有提供如 `ok`、`noContent`、`notFound` 等常見的方法，可以用來替代。

以處理 GET 與 POST 請求的方法為例，將程式調整成如下。
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {
    // ...

    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable("id") String productId) {
        var product = productMap.get(productId);
        return product == null
                ? ResponseEntity.notFound().build()
                : ResponseEntity.ok(product);
    }
    
    @PostMapping
    public ResponseEntity<Void> createProduct(@RequestBody Product product) {
        // ...

        URI uri = ServletUriComponentsBuilder
                .fromCurrentRequestUri()
                .path("/{id}")
                .build(Map.of("id", product.getId()));

        return ResponseEntity.created(uri).build();
    }

    // ...
}
```

其中 `ok` 方法可傳入 payload，而 `created` 方法可傳入 URI。

### （三）封裝多個查詢字串
在第二節的實作的 `GET /products` 這支 API，若開放更多 query string 傳進來，將會使程式碼太冗長。畢竟每個 query string，都要配上一個 `@RequestParam` 注解，而每個注解都能傳入好幾個參數。

針對這個問題，我們可準備自定義的類別，並搭配一個叫做 `@ModelAttribute` 的注解，將 query string 的值自動填充到該類別的物件中。就像接收 request body 那樣。

以下是用來接收 query string 的類別。
``` java
public class ProductRequestParameter {
    private String searchKey;
    private String sortField;
    private String sortDir;

    // getter, setter ...
}
```

以下是調整過後的 API 處理方法。
``` java
@RestController
@RequestMapping(path = "/products")
public class ProductController {
    // ...

    @GetMapping
    public ResponseEntity<List<Product>> getProducts(
            @ModelAttribute ProductRequestParameter param
    ) {
        var sortField = param.getSortField();
        var sortDir = param.getSortDir();
        var keyword = param.getSearchKey();

        // ...
    }
}
```

透過 `@ModelAttribute` 注解，可以將 query string 都收集起來，之後再分別取出做運用。

我們在 Controller 寫了好幾支程式，內容也越來越龐大。<a href="/articles/spring-boot-three-tier-architecture" target="_blank">第 4 課</a>將介紹「三層式架構」，將不同目的的程式碼片段，分開撰寫在其他地方。

本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch03.2-use-query-string-and-header-in-controller)。