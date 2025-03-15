---
title: 【Spring Boot】第9.2課－使用 JPA 設計實體類別與 MySQL 資料表欄位
date: 2024-05-29 20:00:00
permalink: articles/spring-boot-mysql-column-definition-with-jpa
index_img: /img/index_img/spring-boot.jpg
excerpt: 我們知道可藉由定義實體類別，讓 Spring Data JPA 建立出資料表。而本文會介紹各種設定欄位的方式，包含最基本的欄位名稱、長度與唯一性。此外也會重複運用具有相同設定的欄位，包含嵌入物件與繼承基底類別。最後說明如何自動在欄位填入日期時間與使用者資料。
categories:
  - ["Spring Boot"]
---

在上一篇，我們了解可藉由定義實體類別，讓 Spring Data JPA 建立出資料表。而本文會介紹各種設定資料表欄位的方式，包含欄位名稱、長度與唯一性等。

此外也會重複運用具有相同設定的欄位，包含嵌入物件與繼承基底類別。最後說明如何在插入或更新資料時，自動在欄位填入日期時間與使用者資料。


-----


## 一、實體類別介紹
Spring Data JPA 允許我們在程式中，透過定義實體類別的方式，來設計資料表（table）。所謂的實體類別，指的是要儲存到資料庫的資料類別。

以下建立一個實體類別，描述了學生資料。
``` java
import jakarta.persistence.*;

@Entity
@Table(name = "student")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private int grade;
    private BloodType bloodType;
    private LocalDate birthday;

    // getter, setter ...
}
```

該類別叫做「Student」，目前包含 id、名字、年級、血型與生日，共 5 個欄位。

其中的「BloodType」是個「列舉」（enumeration）類別，包含 4 種血型。
``` java
public enum BloodType {
    A, B, O, AB
}
```

## 二、設計資料表欄位
實體類別會對應到資料庫的 table，因此維護該類別，就相當於維護 table。在實體類別設計欄位時，會搭配使用 @Column 注解，透過它的參數進行設定。
|參數|設定項目|補充說明|
|-|-|-|
|name|欄位名稱|針對同一個欄位，我們可以在實體類別與 table，分別使用不同的名稱。|
|length|字串長度|超出的部份會被截斷（truncate）。|
|nullable|值是否可為 null|Java 基本型態預設為 false；參考型態（如 String）預設為 true。|
|unique|值是否唯一|若為 true，會自動建立「唯一索引」。|
|precision|整數與小數的總位數|適用於 Java 的 BigDecimal 型態；MySQL 的 DECIMAL 型態。|
|scale|小數在 precision 所佔的位數|適用於 Java 的 BigDecimal 型態；MySQL 的 DECIMAL 型態。|

以下是將 `@Column` 注解用在實體類別上。
``` java
@Entity
@Table(name = "student")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "name", length = 30, unique = true, nullable = false)
    private String name;

    @Column(name = "grade")
    private int grade;

    @Column(name = "blood_type")
    @Enumerated(EnumType.STRING)
    private BloodType bloodType;

    @Column(name = "birthday", nullable = false)
    private LocalDate birthday;

    // getter, setter ...
}
```

另外，如果欄位型態是一個「列舉」（Enum），不妨冠上 `@Enumerated` 注解，並設定以 Enum 值的名字來儲存。否則預設會以序數（ordinal）來儲存，可讀性較差。

在網路上其他文章中，讀者可能會注意到那些範例程式，即便實體類別的欄位名與 table 欄位一致，依然會再度使用 `@Column` 注解來設定 table 欄位名稱。筆者認為這樣的好處，是能避免日後在實體類別修改欄位名稱，無意間影響與 table 的對應關係。

設定完成後，請讀者啟動程式，讓 Spring Data JPA 自動在資料庫產生 table。下圖是在 MySQL 使用 `DESCRIBE` 指令，確認 table 的定義。
<img src="{{ permalink }}mysql-workbench-describe-table-with-more-fields.png" />

在結果中，可看到欄位型態、是否為唯一欄位，或者是否允許 null 值等資訊。

## 三、嵌入物件欄位
我們也可透過「嵌入」（embed）的方式，將自定義的物件放入實體類別中。

以下建立叫做「Contact」的類別，描述了聯繫方式。
``` java
@Embeddable
public class Contact {
    private String email;
    private String phone;

    // getter, setter ...
}
```

此類別包含信箱與電話這 2 個欄位。為了將它嵌入到實體類別中，需冠上 `@Embeddable` 注解。這麼做的好處，是日後若有其他實體類別（如老師、員工、公司等），均可重複使用。

接著回到 Student 實體類別，使用 `@Embedded` 注解嵌入 Contact 類別。
``` java
@Entity
@Table(name = "student")
public class Student {
    // ...

    @Embedded
    @AttributeOverride(name = "email", column = @Column(name = "contact_email"))
    @AttributeOverride(name = "phone", column = @Column(name = "contact_phone"))
    private Contact contact;

    // getter, setter ...
}
```

為了在實體類別對嵌入的物件欄位進行設定，需使用 `@AttributeOverride` 注解。它具有兩個參數，name 是指向內部的欄位；而 `column` 則是傳入前面介紹過的 `@Column` 注解，進行設定。

## 四、自動填入日期時間與使用者
### （一）前言
我們可能會想記錄每一筆資料的異動資訊。比方說文章是何時發表的、何時修改的、誰建立的、誰更新的。透過 Spring Data 提供的「JPA Auditing」功能，可以幫助我們做這件事。

請先在啟動類別冠上 `@EnableJpaAuditing` 注解，以啟用此功能。
``` java
@SpringBootApplication
@EnableJpaAuditing
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

接著在實體類別類別冠上 `@EntityListeners` 注解，註冊這個功能。
``` java
@Entity
@Table(name = "student")
@EntityListeners(AuditingEntityListener.class)
public class Student {
    // ...
}
```

### （二）填入日期時間
以下在 Student 實體類別中，添加了 2 個欄位，分別代表資料的建立與更新時間。
``` java
@Entity
@Table(name = "student")
@EntityListeners(AuditingEntityListener.class)
public class Student {
    // ...
    
    @Column(name = "created_time", nullable = false)
    @CreatedDate
    private LocalDateTime createdTime;

    @Column(name = "updated_time", nullable = false)
    @LastModifiedDate
    private LocalDateTime updatedTime;

    // getter, setter ...
}
```

只要在欄位分別冠上 `@CreatedDate` 和 `@LastModifiedDate` 注解即可。Spring Data JPA 會在插入或更新資料時，將日期時間的值賦予給這些欄位。

當插入資料時，具有 `@CreatedDate` 或 `@LastModifiedDate` 注解的欄位，都會同時被賦予值。而後續更新資料時，只會刷新 `@LastModifiedDate` 的欄位。

### （三）填入使用者
以下在 Student 實體類別中，添加了 2 個欄位，分別代表資料的建立者與更新者。
``` java
@Entity
@Table(name = "student")
@EntityListeners(AuditingEntityListener.class)
public class Student {
    // ...
    
    @Column(name = "created_by", nullable = false)
    @CreatedBy
    private String createdBy;

    @Column(name = "updated_by", nullable = false)
    @LastModifiedBy
    private String updatedBy;

    // getter, setter ...
}
```

接著在欄位分別冠上 `@CreatedBy` 或 `@LastModifiedBy` 注解。Spring Data JPA 會在插入或更新資料時，將代表使用者的值賦予給這些欄位。

那麼使用者的資料從哪裡來呢？這時我們需要準備一個實作 `AuditorAware` 介面的元件，讓它告訴 Spring Data JPA。
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

這個 `AuditorAware` 介面接收一個泛型類別，它需要與那些冠上 `@CreatedBy` 和 `@LastModifiedBy` 注解的欄位型態相同。由於實體類別中的 createdBy 和 updatedBy 欄位是字串，因此泛型類別應傳入 String。

接著覆寫 `getCurrentAuditor` 方法，它會在資料正要被儲存時，由 Spring Data JPA 觸發。其回傳值會被賦予給具有這兩種注解的欄位。此處回傳隨機字串，當作使用者的 id。

附帶一提，若讀者所開發的專案有引進 Spring Security，那就可以從「Security Context」獲取當前使用者的資訊。

## 五、共享相同的欄位設定
### （一）用於嵌入欄位
在本文第三節，我們在 Student 實體類別嵌入自定義的 Contact 類別，代表聯繫方式。

Spring Data JPA 並不支援直接在 Contact 類別內部使用 `@Column` 注解來設定。因此若將 Contact 也嵌入到其他實體類別，且內部欄位的設定均相同，那麼就會持續使用 `@AttributeOverride` 注解，寫出重複的設定。

但重複利用欄位設定這個想法，還是可以實現的。我們首先需建立一個父類別，將要共享的欄位抽離過去。此外，為了強調該父類別是「基底類別」，不會被用來建立物件，故宣告為抽象。
``` java
@MappedSuperclass
public abstract class BaseContact {
    
    @Column(name = "contact_email", length = 50)
    private String email;

    @Column(name = "contact_phone", length = 50)
    private String phone;
    
    // getter, setter ...
}
```

該基底類別冠上了 `@MappedSuperclass` 注解，用途是讓子類別繼承後，能將父類別的欄位也連動到資料庫的 table 欄位上。
``` java
@Embeddable
public class Contact extends BaseContact {
}
```

如此一來，在 Student 實體類別就不必透過 `@AttributeOverride` 注解來設定欄位值了。除非有少數特例需個別設定，否則可將其移除。

### （二）用於實體類別
這個 `@MappedSuperclass` 注解，亦可用於那些實體類別都具備的欄位，例如 id、建立與更新資料的時間、建立與更新資料的人。

以下針對實體類別建立了一個基底類別。
``` java
@MappedSuperclass
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "created_time", nullable = false)
    @CreatedDate
    private LocalDateTime createdTime;

    @Column(name = "updated_time", nullable = false)
    @LastModifiedDate
    private LocalDateTime updatedTime;

    @Column(name = "created_by", nullable = false)
    @CreatedBy
    private String createdBy;

    @Column(name = "updated_by", nullable = false)
    @LastModifiedBy
    private String updatedBy;

    // getter, setter ...
}
```

而實體類別只要留下自身特有的欄位，並繼承該基底類別即可。一旦基底類別的設計有改動，則所有實體類別都會生效。
``` java
@Entity
@Table(name = "student")
@EntityListeners(AuditingEntityListener.class)
public class Student extends BaseEntity {
    // ...
}
```

本文介紹了如何透過實體類別，設計資料庫中 table 的欄位，讀者可透過重啟程式，印證 table 的變化。

我們尚未印證本文第三節的 `AuditorAware` 元件，在插入或更新資料時的效果。下一篇將使用 Spring Data JPA 提供的介面，實際進行 CRUD。

本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.2-mysql-column-definition-with-jpa)。