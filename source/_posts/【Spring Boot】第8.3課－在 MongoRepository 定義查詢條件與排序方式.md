---
title: 【Spring Boot】第8.3課－在 MongoRepository 定義查詢條件與排序方式
date: 2019-06-23 08:37:56
permalink: articles/spring-boot-mongo-repository-customize-query
index_img: /img/index_img/spring-boot.jpg
excerpt: 本文將使用 Spring Data 框架的功能，示範在 MongoRepository 中，透過方法名稱及原生語法，設計自己的的查詢條件。接著說明如何進行排序與分頁。最後簡介 MongoTemplate，藉由動態產生查詢條件，以因應多樣的需求。
categories:
  - ["Spring Boot"]
---

上一篇已經介紹過 MongoRepository 內建的查詢方法，也就是用 id 欄位來查詢。但資料勢必有其他欄位，只有 id 能做為條件肯定是不夠的。

本文會使用 Spring Data 框架的功能，示範如何在 repository 中設計自己的查詢條件，包含透過方法名稱及原生語法。接著說明如何進行排序與分頁。

最後簡介 MongoTemplate，藉由動態產生查詢條件，以因應多樣的需求。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch08.2-mongodb-repository-crud)。

## 一、資料類別
首先讓我們快速回顧上一篇要儲存到資料庫的學生資料類別。
``` java
@Document(collection = "student")
public class Student {

    @Id
    private String id;
    private String name;
    private int grade;
    private LocalDate birthday;
    private Contact contact;
    private List<Certificate> certificates;
    
    // getter, setter ...
}
```

而以下分別是學生的「聯繫方式」與「證照」，屬於學生資料的內部欄位。
``` java
public class Contact {
    private String email;
    private String phone;

    public static Contact of(String email, String phone) {
        var c = new Contact();
        c.email = email;
        c.phone = phone;
        return c;
    }

    // getter, setter ...
}
```
``` java
public class Certificate {
    private String type;
    private Integer score;
    private String level;

    public static Certificate of(String type, Integer score, String level) {
        var c = new Certificate();
        c.type = type;
        c.score = score;
        c.level = level;
        return c;
    }

    // getter, setter ...
}
```
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
}
```

## 二、自定義查詢方法
我們能在 StudentRepository 中，依照特定的命名規則，設計自己的查詢方法。

本節挑選一部份出來示範。更多命名方式，讀者可參考 [Spring Data JPA 官方文件](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)。雖然「JPA」是針對關聯式資料庫，但裡面的內容仍適用於 MongoDB。

### （一）相等條件
以下是查詢「name」欄位值等於參數值的方法。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
    Student findByName(String s);
}
```

方法名稱中，在 `findBy` 後面緊接著欄位名稱。該名稱必須存在於文件的資料類別，也就是 Student 類別中，否則 Spring Boot 會啟動失敗。

以下是查詢內部欄位的方法，分別是將聯繫方式與證照當作條件。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
    Student findByContactEmail(String s);
    Student findByContactPhone(String s);
    List<Student> findByCertificatesType(String s);
}
```

想在方法名稱中指定內部欄位，只要將欄位的「路徑」寫出即可。第 1、2 個方法分別是查詢 Contact 物件欄位中的 email 與 phone 欄位。

而第 3 個方法的查詢條件，則是在 Certificates 的 List 中，只要任一元素的 type 欄位值等於參數，就算符合。

至於回傳值型態，若讀者認為該條件可能會有多筆資料符合，那就可以選擇回傳 List。

附帶一提，方法的參數名稱，與資料的欄位名稱並無關聯，所以不一定要取相同的名字。

### （二）範圍條件
以下是針對數值與日期欄位，做範圍條件的查詢。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
    List<Student> findByGradeGreaterThanEqual(int from);
    List<Student> findByGradeLessThanEqual(int to);
    
    List<Student> findByBirthdayAfter(LocalDate from);
    List<Student> findByBirthdayBefore(LocalDate to);
}
```

第 1、2 個方法，分別是查詢年級大於等於，或小於等於參數的資料。方法名稱中使用了 `GreaterThanEqual` 和 `LessThanEqual` 關鍵字。若範圍只想要大於或小於，則拿掉 `Equal` 關鍵字即可。

第 3、4 個方法，分別是查詢生日在某個日期之後或之前的資料。方法名稱中使用了 `After` 和 `Before` 做為關鍵字。

### （三）組合多個條件
查詢條件也可透過「AND」或「OR」的邏輯組合起來。

以下的條件是 Contact 物件欄位中的 phone 欄位等於某個值，或 email 欄位等於某個值。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
    List<Student> findByContactEmailOrContactPhone(String email, String phone);
}
```

方法名稱中使用了 `Or` 和 `And` 關鍵字。

要注意的是，透過 AND 邏輯組合查詢條件時，欄位不可重複，否則呼叫時會發生例外。舉例來說，「年級大於等於且小於等於」這項條件，不可寫成：
``` java
List<Student> findByGradeGreaterThanEqualAndGradeLessThanEqual(int from, int to);
```

原因是 Spring Data 會將方法名稱解讀成類似下面的條件。
``` json
{
    "grade": {
        "gte": 1
    },
    "grade": {
        "lte": 4
    }
}
```

在 JSON 資料中，若同一階層有相同名稱的欄位，是不合法的格式。

回到範圍查詢，此時可使用 `Between` 關鍵字，並傳入 `Range` 型態的參數。
``` java
List<Student> findByGradeBetween(Range<Integer> range);
```

而 `Range` 物件的建立方式為：
``` java
Range<Integer> range = Range.of(
    Range.Bound.inclusive(from),
    Range.Bound.inclusive(to)
);
```

### （四）原生語法
讀者也許會覺得上述「findByContactEmailOrContactPhone」的方法名稱有點冗長。或者有些比較複雜的條件，其實很難寫出方法名稱，甚至根本寫不出來。

針對這個情形，Spring Data 也支援我們直接撰寫原生語法，而方法可隨意取名。以下的例子是將相同的查詢條件，以 MongoDB 的語法寫出。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {

    @Query("""
        {
            "$or": [
                { "contact.email": ?0 },
                { "contact.phone": ?1 }
            ]
        }
    """)
    List<Student> findByContact(String email, String phone);
}
```

使用 `@Query` 注解，可以用字串的形式提供語法。而當中的「?0」、「?1」等符號，代表要取用方法的第幾個參數（位置從 0 開始算）。

此外，Spring Data 並不在意語法是否換行與縮排，讀者可自行排版。

以下再提供一個例子，其條件是具有某個證照，且分數大於等於某值，例如多益分數大於等於 900。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
    
    @Query("""
        {
            "certificates": {
                "$elemMatch": {
                    "type": ?0,
                    "score": { "$gte": ?1 }
                }
            }
        }
    """)
    List<Student> findByCertificateTypeAndScoreGte(String type, int score);
}
```

要注意的一點是，假設讀者所開發的系統，未來更換資料庫了，那這些原生語法可能就得根據資料庫的種類進行重寫。而使用方法名稱定義查詢條件，則不必擔心這個問題。

## 三、排序與分頁
### （一）排序
以下是根據年級做遞減排序，而沒有查詢條件。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
    List<Student> findAllByOrderByGradeDesc();
}
```

這是一種簡易的做法，直接在方法名稱加上 `OrderBy` 關鍵字，再緊接著欄位名稱與排序方向。

若想要更彈性一點，可選擇在方法傳入 `Sort` 型態的參數。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {

    @Query("{}")
    List<Student> find(Sort sort);
}
```

在建立 `Sort` 物件時，能夠設定排序的欄位與方向。為了示範用法，以下在 Controller 準備了 API，透過 query string 接收排序方式。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;
    
    @GetMapping("/students")
    public ResponseEntity<List<Student>> getStudents(
            @RequestParam(required = false) String sortField,
            @RequestParam(required = false) String sortDirection
    ) {
        Sort sort = createSort(sortField, sortDirection);
        List<Student> students = studentRepository.find(sort);

        return ResponseEntity.ok(students);
    }
    
    private Sort createSort(String field, String direction) {
        if (field == null && direction == null) {
            return Sort.unsorted();
        }

        if (field == null ^ direction == null) {
            return Sort.unsorted();
        }

        Sort.Order order;
        if ("asc".equalsIgnoreCase(direction)) {
            order = Sort.Order.asc(field);
        } else if ("desc".equalsIgnoreCase(direction)) {
            order = Sort.Order.desc(field);
        } else {
            return Sort.unsorted();
        }

        return Sort.by(List.of(order));
    }
}
```

排序的欄位與方向必須一起提供，或兩者都不提供。若有不合理的值，則呼叫 `Sort.unsorted` 方法，視為不排序。

呼叫 `Sort.Order.asc` 或 `Sort.Order.desc` 方法，並傳入欄位名稱，分別可建立遞增或遞減的排序方式，其型態為 `Order`。

將一至多個 `Order` 物件傳入 `Sort.by` 方法，可得到 `Sort` 物件。它代表整體的排序規則，實現多重排序也不成問題。

### （二）分頁
當資料量太多，實務上會利用「分頁」（pagination）的做法，分批從資料庫取得資料，避免造成資料庫、應用程式或傳輸上的負擔。

分頁通常會與排序一起使用。例如先將資料根據名字欄位做遞增排序，接著每 10 筆資料當作一頁。我們可選擇要取第 1 頁（第 1 ~ 10 筆）、第 2 頁（第 11 ~ 20 筆），依此類推。

要透過 repository 的方法進行排序與分頁，需在方法傳入 `Pageable` 型態的參數。
``` java
@Repository
public interface StudentRepository extends MongoRepository<Student, String> {
    
    @Query("{}")
    List<Student> find(Pageable pageable);
}
```

`Pageable` 是一個介面，而 Spring Data 內建了叫做 `PageRequest` 的實作類別。

呼叫 `PageRequest.of` 方法，依序傳入「第幾頁」、「每頁筆數」，以及前面提到的 `Sort` 物件，就能建立出 `PageRequest` 物件。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;
    
    @GetMapping("/students")
    public ResponseEntity<List<Student>> getStudents(
            @RequestParam(required = false) String sortField, @RequestParam(required = false) String sortDirection,
            @RequestParam(required = false) Integer page, @RequestParam(required = false) Integer size
    ) {
        Sort sort = createSort(sortField, sortDirection);
        Pageable pageable = createPageable(page, size, sort);
        List<Student> students = studentRepository.find(pageable);

        return ResponseEntity.ok(students);
    }
    
    private Pageable createPageable(Integer page, Integer size, Sort sort) {
        if (page == null && size == null) {
            return Pageable.unpaged(sort);
        }

        if (page == null ^ size == null) {
            return Pageable.unpaged(sort);
        }

        return PageRequest.of(page, size, sort);
    }
    
    private Sort createSort(String field, String direction) {
        // ...
    }
}
```

分頁的頁數與每頁筆數，同樣也必須一起提供，或兩者都不提供。若有不合理的值，則呼叫 `Pageable.unpaged` 方法，視為不分頁。

## 四、因應變化多端的查詢條件
### （一）背景
看完這篇文章後，我們在 repository 中設計了許多方法。不知讀者是否有察覺到，實務上面對不同的需求，就會有各式各樣的查詢條件，並且有不同的排序與分頁方式。

舉例來說，假設有一支 API 是 `GET /students`，用途是查詢多筆資料，那它可能會支援大量的 query string。例如：信箱、手機、年級範圍、生日範圍、證照種類及其分數等。以下為示意程式。
``` java
@RestController
public class MyController {

    @GetMapping("/students")
    public ResponseEntity<List<Student>> getStudents(@ModelAttribute StudentRequestParam param) {
        // ...
    }
}
```

此處將可能收到的 query string 封裝成如下的物件：
``` java
public class StudentRequestParam {
    // sort and pagination
    private String sortField;
    private String sortDirection;
    private Integer page;
    private Integer size;
    
    // customized parameters
    private String email;
    private String phone;
    private Integer gradeFrom;
    private Integer gradeTo;
    private LocalDate birthdayFrom;
    private LocalDate birthdayTo;
    private String certType;
    private Integer certScoreFrom;
    private Integer certScoreTo;

    // getter, setter ...
}
```

當查詢條件複雜一點，前端就會傳遞各種 query string 的組合給 API。若後端總是 case by case，在 repository 設計新方法來因應，並在程式中判斷要呼叫哪一個 repository 方法，那會沒完沒了。

在商業邏輯中，透過 Spring Data 的 repository 進行簡單的查詢是很便利的。然而後端若想兼容各種 query string 的組合，條件寫死的 repository 方法有時並不好用。

### （二）MongoTemplate 簡介
根據筆者在前公司的經驗，會使用 `MongoTemplate` 元件，並搭配動態產生的條件進行複雜查詢，以下是一些用法。

假設查詢條件是「2 年級的學生」，且不做排序與分頁，則示意程式如下：
``` java
@RestController
public class MyController {

    @Autowired
    private MongoTemplate mongoTemplate;

    private List<Student> foo() {
        Criteria criteria = Criteria.where("grade").is(2);
        Pageable pageable = Pageable.unpaged();
        Query query = new Query(criteria).with(pageable);

        return mongoTemplate.find(query, Student.class);
    }
}
```

假設查詢條件是「多益分數達 900 分的 1 年級學生」，則示意程式如下：
``` java
@RestController
public class MyController {

    @Autowired
    private MongoTemplate mongoTemplate;

    private List<Student> foo() {
        Criteria certCriteria = Criteria
                .where("certificates")
                .elemMatch(
                        new Criteria().andOperator(
                                Criteria.where("type").is("TOEIC"),
                                Criteria.where("score").gte(900)
                        )
                );
        Criteria gradeCriteria = Criteria.where("grade").is(1);
        Criteria allCriteria = new Criteria().andOperator(List.of(gradeCriteria, certCriteria));
        Query query = new Query(allCriteria);

        return mongoTemplate.find(query, Student.class);
    }
}
```

從範例中可看出，透過 Criteria 物件，能夠以程式碼動態建立出查詢條件。

筆者在「[【Elasticsearch】使用 Java API Client 完成簡易搜尋框架](https://ithelp.ithome.com.tw/articles/10339546)」文章中，有刻過一套動態產生條件的查詢流程。以下提供當時的實作思路，供讀者參考：
1. 將所有 query string 封裝成一個物件，並透過 Java 的「反射」（reflection）轉換成泛用的 Map 資料結構。
2. 處理排序與分頁的 query string。
3. 處理需「客製化」的 query string，如範圍條件、內部欄位條件。
4. 其餘 query string 視為欄位的相等條件。
5. 將所有條件以 AND 邏輯串接起來。
6. 執行查詢。

本節只是簡單介紹用法。至於 `MongoTemplate` 元件的配置方式，以及要如何開發出通用且易於擴充的查詢流程，讀者得自行去打造。

本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch08.3-mongo-repository-customize-query)。