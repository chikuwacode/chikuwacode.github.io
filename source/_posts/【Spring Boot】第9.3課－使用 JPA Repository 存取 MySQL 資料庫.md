---
title: 【Spring Boot】第9.3課－使用 JPA Repository 存取 MySQL 資料庫
date: 2024-05-30 08:30:00
permalink: articles/spring-boot-mysql-using-jpa-repository
index_img: /img/index_img/spring-boot.jpg
excerpt: 為了在 Spring Boot 專案中存取 MySQL 資料庫，我們可借助 Spring Data JPA 提供的 repository 介面。本文除了透過內建的 CRUD 方法進行存取，也會設計自己的查詢條件，包含透過方法名稱及原生語法。最後說明如何排序與分頁。
categories:
  - ["Spring Boot"]
---

為了在 Spring Boot 專案中存取 MySQL 資料庫，我們可借助 Spring Data JPA 框架所提供的 repository 介面。

本文除了透過內建的 CRUD 方法進行存取，也會設計自己的查詢條件，包含透過方法名稱及原生語法。最後說明如何排序與分頁。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.2-mysql-column-definition-with-jpa)。

## 一、實體類別介紹
讓我們快速回顧上一篇的實體類別。以下的「Student」類別描述了學生資料，會儲存到資料庫中。
``` java
@Entity
@Table(name = "student")
@EntityListeners(AuditingEntityListener.class)
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "grade")
    private int grade;

    @Column(name = "blood_type")
    @Enumerated(EnumType.STRING)
    private BloodType bloodType;

    @Column(name = "birthday", nullable = false)
    private LocalDate birthday;
    
    @Enumerated(EnumType.STRING)
    private BloodType bloodType;
    
    @Embedded
    @AttributeOverride(name = "email", column = @Column(name = "contact_email"))
    @AttributeOverride(name = "phone", column = @Column(name = "contact_phone"))
    private Contact contact;
    
    @CreatedDate
    private LocalDateTime createdTime;

    @LastModifiedDate
    private LocalDateTime updatedTime;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    // getter, setter ...
}
```

以下的「BloodType」是個列舉類別，包含 4 種血型。
``` java
public enum BloodType {
    A, B, O, AB
}
```

以下的「Contact」類別是描述聯繫方式，包含信箱與電話這 2 個欄位。
``` java
@Embeddable
public class Contact {
    private String email;
    private String phone;

    // getter, setter ...
}
```

以下是<a href="/articles/spring-boot-mysql-column-definition-with-jpa/" target="_blank">上一篇</a>提到的 `AuditorAware` 元件。用途是當插入或更新資料時，能在實體類別中具有 `@CreatedBy` 或 `@LastModifiedBy` 注解的欄位，自動填入使用者資訊，此處以隨機字串代替。
``` java
@Component
public class AuditorAwareImpl implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        String randomId = UUID.randomUUID().toString();
        return Optional.of(randomId);
    }
}
```

## 二、認識 Spring Data 的 Repository 介面
### （一）建立 Repository 介面
在<a href="/articles/spring-boot-bean-ioc-di-and-swap/" target="_blank">第 5 課</a>，我們有練習過建立一個「ProductRepository」介面，將其注入到商業邏輯中。還提供兩種實作類別，分別用 List 與 Map 結構來儲存範例資料。

本節會使用一個特殊的介面，它是由 Spring Data JPA 所提供，其定位與上述的 ProductRepository 相同。但我們不必親自實作它，而是交給框架處理。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
}
```

我們建立了叫做「StudentRepository」的介面，並繼承 `JpaRepository` 介面。繼承時，需在泛型分別傳入實體類別與主鍵類別。

### （二）查詢語法與物件對映
使用 Spring Data 時，我們最直接感受到的好處，就是「產生查詢語法」與「物件對映」。

在 `JpaRepository` 介面，以及它的父介面中，已經有內建一些基本方法。舉例來說，我們可以將 Student 物件傳入 repository 的 `save` 方法。

僅僅一個方法呼叫，Spring Data 就會產生類似下面的 MySQL 指令：
``` sql
INSERT INTO `student` (`name`, `grade`, `blood_type`, `birthday`)
VALUE ("Vincent", 4, "A", "1996-01-01");
```

此時該方法會回傳插入成功的資料，而且會包含由 MySQL 產生的 id。

又或者是呼叫 `findById` 方法，則會產生如下的指令：
``` sql
SELECT *
FROM `student`
WHERE `id` = 1;
```

此時該方法除了回傳符合條件的結果，更重要的是資料會被轉換成 Java 物件，也就是 Student。資料庫中的資料，與程式物件互相轉換，這項技術在關聯式資料庫稱為「物件關聯對映」（Object-Relational Mapping，ORM）。

從上面這兩個例子，我們可看出 Spring Data 會解讀 repository 的方法名稱、傳入參數，以及回傳值型態，產生對應的資料庫語法。

這段過程是透過底層的 ORM 框架來完成，Spring Data JPA 預設是採用「Hibernate」。

## 三、使用 JPA Repository 進行 CRUD
本節讓我們來實際使用 JPA Repository 提供的內建方法。

為了在程式中有地方呼叫 StudentRepository，讀者可在 Controller 準備 API，屆時便能搭配如 Postman 之類的工具，對後端發送 request。

### （一）插入資料
以下提供一支 API，用來建立學生資料。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;

    @PostMapping("/students")
    public ResponseEntity<Void> createStudent(@RequestBody Student student) {
        student.setId(null);
        studentRepository.save(student);

        URI uri = ServletUriComponentsBuilder
                .fromCurrentRequestUri()
                .path("/{id}")
                .build(Map.of("id", student.getId()));

        return ResponseEntity.created(uri).build();
    }
}
```

呼叫 repository 的 `save` 方法即可。若 Student 物件的 id 欄位值是 null，則視為插入。反之則視為更新。

插入資料時，一律由 MySQL 自行產生 id 值，我們無法自行給定。該方法最後會回傳含有 id 的資料，此處附加到 response header 中，讓 API 回傳。

此時讀者也可順道確認一下，Spring Data JPA 是否有自動在 createdTime、updatedTime、createdBy 與 updatedBy 欄位填入值。

### （二）取得資料
以下是查詢指定 id 的資料，呼叫 `findById` 方法即可。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;

    @GetMapping("/students/{id}")
    public ResponseEntity<Student> getStudent(@PathVariable Long id) {
        Optional<Student> studentOp = studentRepository.findById(id);
        return studentOp.isPresent()
                ? ResponseEntity.ok(studentOp.get())
                : ResponseEntity.notFound().build();
    }
}
```

以下是給予字串 List，查詢多筆指定 id 的資料。呼叫 `findAllById` 方法即可。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;
        
    @GetMapping("/students/ids")
    public ResponseEntity<List<Student>> getStudents(@RequestParam List<Long> idList) {
        List<Student> students = studentRepository.findAllById(idList);
        return ResponseEntity.ok(students);
    }
}
```

攜帶 query string 存取此 API 的方式，示意如下：
``` text
GET /students/ids?idList=111,222,333
```

### （三）更新資料
以下是更新指定 id 的資料。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;

    @PutMapping("/students/{id}")
    public ResponseEntity<Void> updateStudent(
            @PathVariable Long id, @RequestBody Student request
    ) {
        Optional<Student> studentOp = studentRepository.findById(id);
        if (studentOp.isEmpty()) {
            return ResponseEntity.notFound().build();
        }

        Student student = studentOp.get();
        student.setName(request.getName());
        student.setGrade(request.getGrade());
        student.setBloodType(request.getBloodType());
        student.setBirthday(request.getBirthday());
        student.setContact(request.getContact());
        studentRepository.save(student);

        return ResponseEntity.noContent().build();
    }
}
```

在邏輯中，我們先判斷該筆資料是否存在。是的話，就將 request 中的資料更新上去，再呼叫 `save` 方法儲存。由於呼叫該方法時，table 已有該 id 的資料，因此會執行更新操作，而非插入。

### （四）刪除資料
以下是刪除指定 id 的資料，呼叫 `deleteById` 方法即可。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;

    @DeleteMapping("/students/{id}")
    public ResponseEntity<Void> deleteStudent(@PathVariable Long id) {
        studentRepository.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

其他內建方法尚有 `saveAll`、`deleteAllById`、`existsById`、`count` 等，讀者可自行探索。

## 四、自定義查詢條件
我們能在 repository 中，依照特定的方法命名規則，設計自己的查詢條件。

筆者在<a href="/articles/spring-boot-mongo-repository-customize-query/" target="_blank">第 8.3 課</a>已經設計過各種查詢 MongoDB 的方法，以及排序的方式。雖然是另一款資料庫，但 Spring Data 提供的 repository，其使用方式大致都相同。

本節僅挑選一部份做快速的示範。更多命名方式，讀者可參考 Spring Data JPA 官方文件。

### （一）相等條件
以下方法是以名字做為查詢條件。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
    Student findByName(String name);
}
```

方法名稱中，在 `findBy` 關鍵字後面緊接著實體類別的欄位名稱即可。

以下方法是查詢內部欄位，分別是將聯繫方式的信箱與電話當作條件。只要在方法名稱將欄位的「路徑」寫出即可。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
    Student findByContactEmail(String email);
    Student findByContactPhone(String phone);
}
```

### （二）範圍條件
以下 2 個方法，分別是查詢年級大於等於，和小於等於某個值的資料。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
    List<Student> findByGradeGreaterThanEqual(int from);
    List<Student> findByGradeLessThanEqual(int to);
}
```

數值欄位可使用 `GreaterThanEqual`、`GreaterThan`、`LessThanEqual` 與 `LessThan` 關鍵字。

以下 2 個方法，分別是查詢生日在某天之後，和之前的資料。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
    List<Student> findByBirthdayAfter(LocalDate from);
    List<Student> findByBirthdayBefore(LocalDate to);
}
```

日期欄位是使用 `After` 與 `Before` 關鍵字。

### （三）組合多個條件
查詢條件可透過 `And` 或 `Or` 的邏輯組合起來。

以下的條件，是聯繫方式的信箱等於某個值，或電話等於某個值。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {
    List<Student> findByContactEmailOrContactPhone(String email, String phone);
}
```

### （四）原生語法
Spring Data 也支援我們直接撰寫原生語法，而方法可隨意取名。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {

    @Query(
            nativeQuery = true,
            value = """
                SELECT *
                FROM `student`
                WHERE `contact_email` = ?1 OR `contact_phone` = ?2
            """
    )
    List<Student> findByContact(String email, String phone);
}
```

使用 `@Query` 注解，可以傳入字串提供語法。而當中的「?1」、「?2」等符號，代表要取用方法的第幾個參數（位置從 1 開始算）。並且 `nativeQuery` 參數需給予 true 值，代表這是原生語法。

要注意的是，SQL 語法結尾請不要加分號 ;，避免被認為語法有誤。例如上述的「?2」代表第二個參數，但 JPA 會看成「?2;」，並認為我們寫錯了，拋出「Ordinal parameter label was not an integer」的例外訊息。

又或者是本文第五節的排序，假設原生語法寫成 `SELECT * FROM student;`，則 JPA 會將排序語法直接附加在後，變成：
``` sql
SELECT *
FROM `student`;
ORDER BY `grade` ASC
```

此時底層的 Hibernate 在執行時，勢必會拋出語法錯誤的例外。

## 五、排序與分頁
### （一）排序
要透過 repository 的方法進行排序，首先得建立 `Sort` 物件。
``` java
@RestController
public class MyController {
    // ...

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

排序的欄位與方向必須一起提供，或兩者都不提供。若有不合理的值，便呼叫 `Sort.unsorted` 方法，視為不排序。

呼叫 `Sort.Order.asc` 或 `Sort.Order.desc` 方法，傳入欄位名稱，分別可建立遞增或遞減的排序方式，其型態為 `Order`。

將一至多個 `Order` 物件傳入 `Sort.by` 方法，可得到 `Sort` 物件。它代表整體的排序規則，實現多重排序也不成問題。

建立出 `Sort` 物件後，傳入 repository 的 `findAll` 方法即可。
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
        List<Student> students = studentRepository.findAll(sort);

        return ResponseEntity.ok(students);
    }
    
    private Sort createSort(String field, String direction) {
        // ...
    }
}
```

### （二）分頁
當資料量太多，實務上會在排序之後，藉由「分頁」（pagination）的做法，分批從資料庫取得資料，避免造成系統負擔。

要透過 repository 的方法進行排序與分頁，需在方法傳入 `Pageable` 型態的參數，提供分頁的方式。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {

    @Query(
            nativeQuery = true,
            value = "SELECT * FROM `student`"
    )
    List<Student> find(Pageable pageable);
}
```

以下是建立 `Pageable` 物件的方式。
``` java
@RestController
public class MyController {
    // ...

    private Pageable createPageable(Integer page, Integer size, Sort sort) {
        if (page == null && size == null) {
            return Pageable.unpaged(sort);
        }

        if (page == null ^ size == null) {
            return Pageable.unpaged(sort);
        }

        return PageRequest.of(page, size, sort);
    }
}
```

這個 `Pageable` 是一個介面，而 Spring Data 內建了叫做 `PageRequest` 的實作類別。

分頁的頁數與每頁筆數必須一起提供，或兩者都不提供。若有不合理的值，便呼叫 `Pageable.unpaged` 方法，視為不分頁。

呼叫 `PageRequest.of` 方法，依序傳入「第幾頁」、「每頁筆數」，以及前面提到的 `Sort` 物件，就能建立出 `PageRequest` 物件。

最後將 `Pageable` 物件傳入 repository 的方法即可。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;
    
    @GetMapping("/students")
    public ResponseEntity<List<Student>> getStudents(
            @RequestParam(required = false) String sortField,
            @RequestParam(required = false) String sortDirection,
            @RequestParam(required = false) Integer page,
            @RequestParam(required = false) Integer size
    ) {
        Sort sort = createSort(sortField, sortDirection);
        Pageable pageable = createPageable(page, size, sort);
        List<Student> students = studentRepository.find(pageable);

        return ResponseEntity.ok(students);
    }
    
    private Sort createSort(String field, String direction) {
        // ...
    }
    
    private Pageable createPageable(Integer page, Integer size, Sort sort) {
        // ...
    }
}
```

本文示範了使用 JPA Repository 對單一 table 進行 CRUD。接下來讓我們進入 2 張 table 的範疇，學習如何配置資料表的關聯。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.3-mysql-using-jpa-repository)。

上一課：<a href="/articles/spring-boot-mysql-column-definition-with-jpa/" target="_blank">【Spring Boot】第9.2課－使用 JPA 設計實體類別與 MySQL 資料表欄位</a>

下一課：<a href="/articles/spring-boot-jpa-one-to-one-relationship/" target="_blank">【Spring Boot】第9.4課－使用 JPA 配置資料表關聯（以一對一關聯為例）</a>