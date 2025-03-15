---
title: 【Spring Boot】第3.1課－在 Controller 實作 RESTful API
date: 2019-05-19 15:16:46
permalink: articles/spring-boot-implement-restful-api-in-controller
index_img: /img/index_img/spring-boot.jpg
excerpt: 後端領域的初學者，應該先具備 Web API 的概念，並且也知道前後端會透過 payload 與 HTTP 狀態碼來交換資料。本文一開始先簡介 MVC 架構。接著在 Spring Boot 中撰寫 RESTful API，將相關概念加以活用。最後會完成 GET、POST、PUT 與 DELETE 這 4 支 API，分別進行 CRUD 操作。
categories:
  - ["Spring Boot"]
---

若讀者是後端領域的初學者，看完<a href="/articles/spring-boot-restful-api" target="_blank">第 2 課</a>後，應該具備 Web API 的概念了，並且也知道前後端會透過 payload 與 HTTP 狀態碼來交換資料。

本文一開始先簡介知名的 MVC 架構，讓讀者知道 Controller 的由來。接著在 Spring Boot 中撰寫 RESTful API，將相關概念加以活用。最後會完成 4 支 API，分別能進行 CRUD。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch01-create-project)。

## 一、MVC 架構簡介
不知讀者以前是否聽過「MVC 架構」？筆者在此簡單地介紹。一套系統，依照職責可區分為 Model、View 與 Controller 三個部份。

其中 View 屬於畫面的呈現，在前後端分離的系統中，View 這塊就是前端的工作了。

另外兩項屬於後端。Model 涵蓋資料處理與存取 DB，範圍最廣。至於 Controller 則負責接收請求，呼叫 Model 進行處理，並將結果回傳給 View。

以生活情境來比喻，就像客人跟櫃台點餐，店員會將菜單送到廚房，之後再將餐點交給我們。此時客人、櫃台店員與廚房，分別為 View、Controller 與 Model。

Controller 接收的請求，內容包括了<a href="/articles/spring-boot-restful-api" target="_blank">第 2 課</a>提到的 query string、payload 與 header。而給予的回應，在本文的實作中，則是將處理結果作為 payload，連同 HTTP 狀態碼一併回傳。

若將原本屬於 Model 職責的資料處理寫在 Controller，或是有些 JSP 專案是把程式碼跟 HTML 寫在一起，那系統越做越大，程式碼就會變得臃腫又混亂。MVC 架構的思想，便是讓程式碼分門別類、各司其職。

### 二、配置 Controller
### （一）測試資料
在實作前，讓我們先準備測試資料，這樣後端才有東西可以回傳。

以下的自訂類別，描述了產品的編號、名稱與價格。
``` java
public class Product {
    private String id;
    private String name;
    private int price;

    public static Product of(String id, String name, int price) {
        var p = new Product();
        p.id = id;
        p.name = name;
        p.price = price;

        return p;
    }

    // getter, setter ...
}
```

### （二）Controller 類別
接著請建立一個類別，筆者取名為 ProductController。在第三節，會在這裡定義跟產品直接相關的 RESTful API。
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

本文沒有串接真實的 DB，作為替代，筆者將測試資料存放在 Java 的 Map 資料結構中。其中 key 為編號，value 為整筆產品資料。

Controller 類別還需加上 `@RestController` 這個「注解」（annotation），告訴 Spring Boot 這是一個 Controller，否則它只是普通的類別。Spring Boot 會使用各式各樣的注解來進行一些定義，讀者在學習過程中將一一認識。

透過 Java 的「反射」（Reflection），Spring Boot 會在啟動時掃描程式專案中的每個類別，以及內部的欄位、方法及其參數，是否具有特定的注解。有的話，便納入管理。此過程稱為「元件掃描」（component scan）。

## 三、實作 RESTful API
### （一）GET 請求（取得資源）
本節讓我們正式在 Controller 實作 API。以下是透過產品編號來取得資料。這個範例的呼叫方式為 `GET /products/B1`。
``` java
@RestController
public class ProductController {
    private static final Map<String, Product> productMap = new HashMap<>();
    // ...

    @GetMapping("/products/{id}")
    public Product getProduct(@PathVariable("id") String productId) {
        var product = productMap.get(productId);
        return product == null ? new Product() : product;
    }
}
```

首先宣告一個叫做 getProduct 的方法，用來處理請求。而它的回傳值 Product 會是 response body 的內容。

接著再冠上 `@GetMapping` 注解，並傳入資源的 endpoint 路徑。我們在 endpoint 上以「{id}」字串來「挖空格」，代表這個位置的值將由前端來提供。

最後看到該方法的「productId」參數，也冠上了 `@PathVariable` 注解，且傳入了「id」字串。代表要將 endpoint 上 {id} 位置的值，賦予給 productId。

這個方法的程式邏輯很簡單，從存放測試資料的 Map 取得資料後回傳即可。若該編號無對應的產品，此處暫時回傳空的內容。

完成後，請啟動 Spring Boot，就能用 Postman 工具試著呼叫看看。
<img src="{{ permalink }}restful-get-product.png" />

確實收到了回應。讀者如果有開發過前端或是 App，且串接過 API，那麼心中應該知道這份 JSON 格式的資料要如何使用了吧！基本上後端的目的就是像這樣回傳內容。

### （二）提供 HTTP 狀態碼
上面的程式略過了一件事，那就是找不到產品時，應該要回傳 404 的 HTTP 狀態碼。

Spring Boot 提供了 `ResponseEntity` 類別物件，讓我們可以自定義 response 的其他地方，而不是只能回傳 payload 而已。以下將進行改寫。
``` java
@RestController
public class ProductController {
    // ...

    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable("id") String productId) {
        var product = productMap.get(productId);
        return product == null
                ? ResponseEntity.status(HttpStatus.NOT_FOUND).build()
                : ResponseEntity.status(HttpStatus.OK).body(product);
    }
}
```

在上面的程式中，透過 `ResponseEntity.status` 方法，可傳入想要的狀態。此處傳入「OK」（200）或「Not Found」（400）。

接著繼續呼叫 `body` 方法，可傳入要回傳的 payload。若無 payload，則呼叫 `build` 方法建立出物件。

最後，getProduct 方法的回傳值要同步修改成 `ResponseEntity`，並於泛型類別提供 payload 的類別。

## 四、處理 POST 請求（建立資源）
將實作 RESTful API 的過程走過一遍後，對於其他種類的請求，我們就能依樣畫葫蘆了。

以下是處理 POST 請求，將 payload 中的內容，與現有的測試資料一起放在 Map 中。
``` java
@RestController
public class ProductController {
    private static final Map<String, Product> productMap = new HashMap<>();
    // ...

    @PostMapping("/products")
    public ResponseEntity<Void> createProduct(@RequestBody Product product) {
        if (product.getId() == null) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).build();
        }

        if (productMap.containsKey(product.getId())) {
            return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).build();
        }

        productMap.put(product.getId(), product);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }
}
```

接收 payload 時，需使用 `@RequestBody` 注解。Spring Boot 會根據 JSON 資料的欄位，自動去呼叫 Product 類別中，其名稱相符的「set」方法，藉此將 payload 的內容填充到 Java 物件上。舉例來說，若偵測 JSON 中有名稱為「name」的欄位，那就會尋找「setName」方法來呼叫。

這個 createProduct 方法的程式邏輯，會先檢查是否有附上產品編號。若無，意味著前端的 payload 有明顯錯誤，故回傳 400 狀態碼（Bad Request）。

接著再檢查 Map 中是否已有相同編號的產品資料。若有，則情境上不合理，故回傳 422 狀態碼（Unprocessable Entity）。

如果資料沒有其他問題，就將它放進 Map 中，並回傳 201 狀態碼（Created），代表成功建立資源。

至於回傳的 `ResponseEntity`，由於筆者在這個範例設計成無 payload，因此泛型類別給予「Void」，代表沒有即可。

## 五、處理 PUT、DELETE 請求
### （一）PUT 請求（更新資源）
以下是處理 PUT 請求，會同時接收 endpoint 上的參數，以及 payload。
``` java
@RestController
public class ProductController {
    private static final Map<String, Product> productMap = new HashMap<>();
    // ...

    @PutMapping("/products/{id}")
    public ResponseEntity<Void> updateProduct(
            @PathVariable("id") String productId, @RequestBody Product request) {
        var product = productMap.get(productId);
        if (product == null) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }

        product.setName(request.getName());
        product.setPrice(request.getPrice());

        return ResponseEntity.status(HttpStatus.NO_CONTENT).build();
    }
}
```

這個方法的程式邏輯，是先檢查 Map 中是否有對應編號的產品。若無，則回傳 404 狀態碼，代表不存在。

如果存在，就將 payload 中的欄位值，覆蓋到原有資料上。

### （二）DELETE 請求（刪除資源）
以下是處理 DELETE 請求，只有接收 endpoint 上的參數而已。
``` java
@RestController
public class ProductController {
    private static final Map<String, Product> productMap = new HashMap<>();
    // ...

    @DeleteMapping("/products/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable("id") String productId) {
        if (!productMap.containsKey(productId)) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }

        productMap.remove(productId);
        return ResponseEntity.status(HttpStatus.NO_CONTENT).build();
    }
}
```

同樣會先檢查是否有對應的資料。若有，就將其從 Map 中移除。

以上的實作都完成後，讀者在 Controller 將擁有四支 RESTful API，能進行 CRUD。

本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch03.1-implement-restful-api-in-controller)。