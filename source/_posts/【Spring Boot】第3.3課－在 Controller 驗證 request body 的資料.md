---
title: 【Spring Boot】第3.3課－在 Controller 驗證 request body 與 query string 的資料
date: 2021-05-26 23:51:00
permalink: articles/spring-boot-validate-request-body-and-query-string
index_img: /img/index_img/spring-boot.jpg
excerpt: 在網頁上填寫資料時，前端會對資料進行檢查，避免送出不合理的值。但若直接對 API 發出請求（如透過 Postman），不就可以繞過檢查了嗎？本文會介紹 @Valid 和許多注解，用來在 Controller 對 request body 與 query string 進行資料驗證。
categories:
  - ["Spring Boot"]
---

在網頁上填寫資料時，前端會對資料進行檢查，避免送出不合理的值。但若直接對 API 發出請求（如透過 Postman），不就可以繞過檢查了嗎？甚至還可能將非預期的值存進資料庫。

本文會介紹許多內建的注解，用來在 Controller 對接收到的 request body 與 query string 進行資料驗證。此外也會示範定義自己的驗證規則。


-----


## 一、什麼是資料驗證
對開發過前端或行動 App 的人來說，使用者填寫完資料送出時，先對資料進行檢查是很正常的事。例如電子郵件地址不允許空白，或者價格不允許負數。

但有心人士只要知道 API 路徑和接受的資料格式，就能透過其他工具或途徑來發出請求（如 Postman）。

那麼 API 是如何得知的呢？舉例來說，讀者在 Chrome 瀏覽器按下 F12，可開啟開發者工具查看。下面是以在網路論壇「Dcard」發表文章為例子。
<img src="{{ permalink }}chrome-devtools-headers-general.png" />
<img src="{{ permalink }}chrome-devtools-payload.png" />

點擊最上方的「Network」頁籤，便能查看該分頁發出的請求。圖中的「Request URL」代表 API 路徑。再往下則是 request 與 response header。點擊「Payload」頁籤，則可看見 request body。

除了有心人士，一般開發者在串接 API 時，也可能不小心在 request 攜帶不合理的資料。如果後端沒有機制來阻擋，就可能影響業務邏輯的程式執行，或是儲存非預期的資料。

然而透過撰寫程式碼的方式來驗證每個資料，其實也不方便。因此，本文將使用注解（annotation）的方式來處理驗證的工作。

## 二、範例專案準備
### （一）Controller 介紹
以下的 Product 與 Price 類別，會做為 request body。
``` java
public class Product {
    private String id;
    private String name;
    private Price price;
    private List<String> categories;
	
	// getter, setter ...
}
```
``` java
public class Price {
    private double amount;
    private String currencyType;
	
	// getter, setter ...
}
```

以下的 BaseParameter 類別，會用來接收 query string。
``` java
public class BaseParameter {
    private String sortField;
    private String sortDirection;
    private Integer page;
    private Integer size;
	
	// getter, setter ...
}
```

以下是 Controller。
``` java
@RestController
@RequestMapping(value = "/products")
public class ProductController {
    
    @PostMapping
    public ResponseEntity<Void> createProduct(@RequestBody Product request) {
        return ResponseEntity.ok().build();
    }
	
	@GetMapping
    public ResponseEntity<List<Product>> getProducts(@ModelAttribute BaseParameter param) {
        return ResponseEntity.ok().build();
    }
}
```

### （二）宣告驗證
我們會在這些接收 request body 與 query string 的類別中，於欄位上使用注解，來定義驗證規則。

請在 pom.xml 檔案添加依賴。
``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

接著將 `@Valid` 注解冠在 API 處理方法的參數前面，代表要對這個資料進行驗證。
``` java
import jakarta.validation.Valid;

@RestController
@RequestMapping(value = "/products")
public class ProductController {
    
    @PostMapping
    public ResponseEntity<Void> createProduct(@Valid @RequestBody Product request) {
        return ResponseEntity.ok().build();
    }
	
	@GetMapping
    public ResponseEntity<List<Product>> getProducts(@Valid @ModelAttribute BaseParameter param) {
        return ResponseEntity.ok().build();
    }
}
```

讀者可能還會找到另一個注解叫 `@Validated`，它是 Spring Boot 自帶的。雖然兩者都能用，但差別在於 `@Validated` 無法冠在欄位上。

這邊提醒一點，`@Valid` 注解只會針對類別中的第一層欄位進行驗證。因此在上述的 Product 類別中，price 欄位的內部資料並不會被驗證。這部份我們留到第七節再來處理。下一節先讓我們一一認識各種驗證注解。

## 三，驗證空與非空
使用 `@NotNull`，可規定該欄位為「必填」，也就是不能為 null 值。
``` java
public class Product {

    @NotNull
    private String name;

    @NotNull
    private Price price;

    @NotNull
    private List<String> categories;
	
	// ...
}
```

讀者可使用 Postman 發出 request，會發現得到 HTTP 400（Bad Request）的狀態碼。
<img src="{{ permalink }}postman-bad-request.png" />

使用 `@NotEmpty`，可規定 String、Collection、Map 與陣列不能是空的，至少要有 1 個字或元素。
``` java
public class Product {

    @NotEmpty
    private String name;

    @NotEmpty
    private List<String> categories;
	
	// ...
}
```

使用 `@NotBlank`，可規定 String 除了要有字，而且不能全部都是半形空白。
``` java
public class Product {

    @NotBlank
    private String name;
	
	// ...
}
```
``` java
public class BaseParameter {

    @NotBlank
    private String sortField;

    @NotBlank
    private String sortDirection;
	
	// ...
}
```

使用 `@Null`，可規定該欄位值必須是 null。
```java
public class Product {

    @Null
    private String id;
	
	// ...
}
```

會用到 `@Null` 的時機，可能是後端想要自行給該欄位賦值。但筆者認為，該類別或許已經身兼多個用途了，比方說 request body 與資料庫的資料類別共用。

若後端能夠設計成將 request body（不含 id 欄位）轉換成另一個類別（含 id 欄位），那就不需要此注解了。

## 四、驗證數值
接下來讓我們繼續認識其他驗證注解。要注意的是，它們都允許欄位值是 null。因此讀者可視需要，搭配 `@NotNull` 注解，確保欄位為必填。

### （一）最大與最小值
使用 `@Max` 與 `@Min`，分別可規定數值欄位的最大與最小值。
``` java
public class Price {
    
    @Min(0)
    @Max(100000)
    private double amount;
	
	// ...
}
```
``` java
public class BaseParameter {

    @NotNull
    @Min(0)
    private Integer page;

    @NotNull
    @Min(1)
    private Integer size;
	
	// ...
}
```

### （二）驗證位數
使用 `@Digits`，可規定數值欄位的整數和小數位數。
``` java
public class Price {
    
    @Digits(integer = 6, fraction = 2)
	@Min(0)
    @Max(100000)
    private double amount;
	
	// ...
}
```

上面限制了欄位值的整數最多能有 6 位，而小數最多 2 位。

## 五、驗證長度
使用 `@Size` 注解，可規定 String、Collection、Map 與陣列的長度或元素數量。
``` java
public class Product {

    @Size(min = 1, max = 3)
    @NotNull
    private List<String> categories;
	
	// ...
}
```

上面限制了欄位值必須要有 1 ~ 3 個元素。

而下面限制了欄位值固定是 3 個字。
``` java
public class Price {

    @Size(min = 3, max = 3)
    @NotNull
    private String currencyType;
	
	// ...
}
```

## 六、正則表達式
使用 `@Pattern` 注解，可使用正則表達式來驗證字串。

以下的例子，是限制欄位值需為 3 個大寫英文字母。
``` java
public class Price {

    @Pattern(regexp = "[A-Z]{3}$")
    @NotNull
    private String currencyType;
	
	// ...
}
```

以下的例子，是限制欄位值只能包含大小寫英文字母、數字與半形空白。
``` java
public class Product {

    @Pattern(regexp = "^[A-Za-z0-9 ]*$")
    @NotBlank
    private String name;
	
	// ...
}
```

## 七、嵌套驗證
### （一）驗證內部物件
在第二節的第二段有提到，`@Valid` 注解只會驗證該類別的第一層欄位。所以剛剛在 Price 類別使用的驗證注解，是不會生效的。

此時讀者只要在想驗證的物件欄位上，也冠上 `@Valid` 注解即可。
``` java
public class Product {

    @Valid
    @NotNull
    private Price price;
	
	// ...
}
```

### （二）驗證元素
如果欄位的型態是 `List<String>`，當我們想驗證 List 中的 String 元素，該怎麼做呢？其實也很簡單，針對 List 的泛型類別冠上驗證注解就好。
``` java
public class Product {

    @Size(min = 1, max = 3)
    @NotNull
    private List<@NotBlank String> categories;
	
	// ...
}
```

再假設我們有如下的類別，其 List 欄位的元素，是我們自定義的物件。
``` java
public class BatchProduct {
    private List<@Valid Product> products;
	
	// getter, setter ...
}
```

只要在泛型類別冠上 `@Valid` 注解，就能驗證 List 裡面的物件元素了。

## 八、自定義驗證邏輯
接下來筆者會介紹更進階的用法。

### （一）建立驗證注解
讓我們以第六節的 currencyType 欄位做延伸。生活中有許多縮寫都是由大寫英文字母組成，例如幣別（如 USD、JPY）、國碼（如 US、JP、KR），或是公司的部門代號。

本節將介紹如何建立自己的驗證注解。一來能夠實作複雜的驗證邏輯，二來也能將多個現有的驗證規則，整合成一個可讀性更好的注解。

以下建立一個叫 @UppercaseAlphabet 的注解，用途是讓被驗證的字串只能包含大寫字母，且可限制字串長度。
``` java
@Constraint(validatedBy = UppercaseAlphabetValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface UppercaseAlphabet {
    int minLength() default 0;
    int maxLength() default Integer.MAX_VALUE;

    String message() default "Should be uppercase alphabet.";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

這邊有幾個重要的地方要知道。
* `@Constraint` 注解：可設定驗證邏輯所在的類別，在本節第二段會介紹。
* minLength、maxLength 參數：這是自定義的參數，冠上注解時可傳入。
* `message` 參數：這是驗證注解必須定義的參數之一，用途是提供驗證未通過的訊息，在第九節會介紹。

### （二）實作驗證過程
在 @UppercaseAlphabet 注解，有冠上了 `@Constraint` 注解。透過它的 `validatedBy` 參數，可提供實作 `ConstraintValidator` 介面的類別。進行資料驗證時，該介面所實作的方法會自動被呼叫。

以下是自定義的驗證邏輯類別。
``` java
public class UppercaseAlphabetValidator implements ConstraintValidator<UppercaseAlphabet, String> {
    private int minLength;
    private int maxLength;

    @Override
    public void initialize(UppercaseAlphabet annotation) {
        this.minLength = annotation.minLength();
        this.maxLength = annotation.maxLength();

        if (this.minLength < 0 || this.minLength > this.maxLength) {
            throw new IllegalArgumentException();
        }
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            value = "";
        }

        if (value.length() < this.minLength || value.length() > this.maxLength) {
            return false;
        }

        return value.matches("^[A-Z]*$");
    }
}
```

在實作 `ConstraintValidator` 介面時，需傳入兩個泛型類別，第一個是自定義的注解，第二個是要驗證的資料型態。

該介面有兩個方法要實作。
* `initialize`：可進行初始化，在此取出注解中的參數備用，並檢查參數是否合理。
* `isValid`：要驗證的資料會被傳入 `value` 參數，我們在此檢查字串的長度，並透過正則表達式確認是否只包含大寫英文字母。

實作完成後，將自定義的注解冠在 request body 的欄位上就行了。下面的例子是限制欄位值為 3 個大寫英文字母。
``` java
public class Price {
    
    @UppercaseAlphabet(minLength = 3, maxLength = 3)
    private String currencyType;
	
	// ...
}
```

讀者也能自行作其他練習，例如將 `@Min` 與 `@Max` 注解合併成一個，允許數值在範圍內。

## 九、驗證未過的訊息
當 `@Valid` 注解驗證失敗時，後端會回傳 HTTP 400 的狀態碼，但我們從 response 一時也看不出未通過的原因。其實從 Console 視窗可找到錯誤訊息。
<img src="{{ permalink }}spring-boot-validation-failed-log.png" />

上圖是「name」和「price.amount」欄位值驗證失敗的 log。這樣的形式除了不易閱讀，而且前端的同事若遇到問題，一直向後端確認原因也很不方便。

本節的目的是將錯誤訊息整理好，並放在如下的 response body 回傳。
``` java
public class ValidationFailInfo {
    private String field;
    private Object value;
    private String message;

    // getter, setter ...
}
```

從 log 可看出 request body 驗證失敗時，會發生 `MethodArgumentNotValidException` 例外。若 query string 驗證失敗，則會發生 `BindException`。

若想捕捉 Controller 所拋出的特定例外，可宣告冠有 `@ExceptionHandler` 注解的方法。
``` java
import org.springframework.validation.BindException;

@RestController
@RequestMapping(value = "/products")
public class ProductController {
	// ...
	
    @ExceptionHandler(BindException.class)
    public ResponseEntity<List<ValidationFailInfo>> handleValidationFail(BindException ex) {
        var infoList = new ArrayList<ValidationFailInfo>();

        ex.getBindingResult().getFieldErrors().forEach(error -> {
            var info = new ValidationFailInfo();
            info.setField(error.getField());
            info.setValue(error.getRejectedValue());
            info.setMessage(error.getDefaultMessage());

            infoList.add(info);
        });

        return ResponseEntity.badRequest().body(infoList);
    }
}
```

關於 `@ExceptionHandler` 注解，在筆者以前的[文章](https://chikuwa-tech-study.blogspot.com/2023/02/spring-boot-controller-advice-handle-exception-and-query-string.html)有專門介紹，本文就不贅述。巧合的是，`MethodArgumentNotValidException` 是 `BindException` 的子類別，所以這邊統一捕捉 `BindException` 即可。

這個方法的實作邏輯，簡單來說就是從捕捉到的例外取出資料做處理，再像 API 一樣回傳 response 給前端。完成後，用 Postman 確認結果如下：
<img src="{{ permalink }}postman-bad-request-with-validation-info-body.png" />

若讀者想定義自己的錯誤訊息，在每個驗證注解都有提供叫 `message` 的參數可以傳入。
``` java
public class Price {

    @Digits(integer = 6, fraction = 2, message = "最多整數6位，小數2位")
    @Min(0)
    @Max(10000)
    private double amount;

    @UppercaseAlphabet(minLength = 3, maxLength = 3, message = "需為3個大寫字母")
    private String currencyType;
	
	// ...
}
```

當我們為 request body 的許多欄位加上驗證規則，並且像這樣子回傳允許的值，某種程度上也是在暴露自己的 API 規格。

如果有安全上的疑慮，筆者認為可以在不同環境（如開發、測試、正式）的 application.properties 配置檔提供設定值，再透過 if 判斷來控制是否回傳錯誤訊息。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch03.3-validate-request-body-and-query-string)。

上一課：<a href="/articles/spring-boot-use-query-string-and-header-in-controller/" target="_blank">【Spring Boot】第3.2課－在 Controller 接收 query string 與操作 header</a>

下一課：<a href="/articles/spring-boot-three-tier-architecture/" target="_blank">【Spring Boot】第4課－實作三層式架構的 Service 與 Repository</a>