---
title: 【Spring Boot】第9.5課－使用 JPA 建立一對多關聯，並配置雙向關聯
date: 2024-06-12 08:30:00
permalink: articles/spring-boot-jpa-one-to-many-relationship-and-bidirectional-association
index_img: /img/index_img/spring-boot.jpg
excerpt: 上一篇已經示範過如何建立兩張資料表的關聯，並接觸加載策略與級聯的知識。本文將以此為基礎，建立一對多關聯，並撰寫 RESTful API 進行測試。過程中也會說明如何在現有的資料上添加 NOT NULL 的關聯。最後則介紹雙向關聯，讓這兩個實體類別，都能取得所關聯的另一方的資料。
categories:
  - ["Spring Boot"]
---

在上一篇，已經示範過如何使用 JPA 建立兩張資料表的一對一關聯，並接觸加載策略與級聯的背景知識。本文將進一步以「學生」就讀「科系」為情境，建立一對多關聯，並撰寫 RESTful API 進行測試。

過程中，也會說明如何針對資料表中的現存資料添加關聯。最後則介紹雙向關聯，讓這兩個實體類別，都能取得所關聯的另一方的資料。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.4-jpa-one-to-one-relationship)。

## 一、程式專案準備
### （一）實體類別介紹
以下的「Student」類別，描述了學生資料，包含 id、名字與聯繫方式，共 3 個欄位。其中聯繫方式是上一篇示範的一對一關聯，在本文不會用到。
``` java
@Entity
@Table(name = "student")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;    
    
    private String name;
    
    @OneToOne(fetch = FetchType.LAZY, cascade = {CascadeType.PERSIST, CascadeType.REMOVE})
    @JoinColumn(name = "contact_id", referencedColumnName = "id", nullable = false, unique = true)
    private Contact contact;
    
    // getter, setter ...
}
```

以下的「Department」類別，描述了科系，包含 id 與名稱這 2 個欄位。
``` java
@Entity
@Table(name = "department")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    
    // getter, setter ...
}
```

由於 Department 也是要儲存到資料庫的實體類別，因此需準備 repository。
``` java
@Repository
public interface DepartmentRepository extends JpaRepository<Department, Long> {
}
```

以下的「Contact」類別，描述了聯繫方式，包含信箱與電話這 2 個欄位。它是來自上一篇的程式碼，因此本文盡量不提及。
``` java
@Entity
@Table(name = "contact")
public class Contact {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String email;
    private String phone;
    
    // getter, setter ...
}
```

### （二）準備測試資料
定義好實體類別後，讀者可啟動程式，讓 Spring Data JPA 建立 table。

接著執行以下 SQL 指令，建立科系的測試資料。畢竟使用情境中，必定是先有科系，才會有學生來就讀。
``` sql
INSERT INTO `department` (`name`)
VALUES ("資訊管理"), ("財務金融"), ("會計");
```

## 二、設計一對多關聯
### （一）設計原則
所謂的一對多關聯，指的是其中一張 table 的 1 筆資料，可以對應到另一張 table 的多筆資料。

現在我們有「學生」與「科系」這兩張 table。每位學生都對應到一個科系，而一個科系可對應到許多學生。事實上，「一對多」和「多對一」是同樣的概念，端看讀者站在哪張 table 的角度。

這兩張 table 要如何關聯起來呢？若讀者學過資料庫的正規化，很直覺一定是在學生表添加額外的欄位做為外鍵（Foreign Key，FK），指向科系表的主鍵（Primary Key，PK）。

畢竟每位學生只會對應到一個科系，在 table 中用一個欄位來存科系 id 是沒問題的。然而科系表的一筆資料是無法儲存多位學生的 id，因此才將科系做為學生資料的一部份。
<img src="{{ permalink }}one-to-many-relationship-student-department.png" />

### （二）程式配置
回到程式專案，以下是在 Student 實體類別配置關聯。
``` java
@Entity
@Table(name = "student")
public class Student {
    // ...
    
    @ManyToOne(fetch = FetchType.LAZY, cascade = {CascadeType.PERSIST})
    @JoinColumn(name = "dept_id", referencedColumnName = "id", nullable = false)
    private Department department;

    // getter, setter ...
}
```

這個 Department 欄位會以 FK 的形式儲存在學生表中，因此需使用 `@JoinColumn` 注解來取名。在 `name` 參數可傳入欄位名稱；在 `referencedColumnName` 參數則定義該欄位要關聯到科系表的 id 欄位，也就是 PK。另外，學生一定會有科系，故設為必填。

`@ManyToOne` 注解的使用方式，與上一篇的一對一關聯相同。我們將加載策略設為 `FetchType.LAZY`，代表不要在查詢學生時，立刻查詢科系。而級聯設為 `CascadeType.PERSIST`，代表插入或更新學生時，能夠在 FK 欄位填入科系編號。

### （三）維護現有資料
若讀者在學生表中已經有一些現有的資料，那麼在啟動程式，讓 JPA 處理 table 的關聯前，需要先知道一件事。

我們雖然在 Student 類別中，有將科系編號的 FK（即 dept_id 欄位）設為必填，但目前學生資料的 FK 欄位並沒有值，違背了必填這項規則。因此啟動程式後，會出現如下的例外訊息：
``` text
java.sql.SQLIntegrityConstraintViolationException:
    Cannot add or update a child row: a foreign key constraint fails
```

如果讀者閱讀本文時是在練習，且願意捨棄現有資料，那麼可以刪除所有學生資料後再重啟。然而在工作上，我們不應該就這麼刪除資料，因此需要手動維護 table。

具體做法是先在學生表新增 FK 欄位，但不設為必填。接著再將科系編號更新上去，最後才將欄位設為必填。示意 SQL 指令如下：
``` sql
ALTER TABLE `student` ADD COLUMN `dept_id` BIGINT;

-- 更新一位學生
UPDATE `student`
SET `dept_id` = (SELECT `id` FROM `department` WHERE `name` = "資訊管理")
WHERE `name` = "Vincent";

ALTER TABLE `student` CHANGE COLUMN `dept_id` `dept_id` BIGINT NOT NULL;
```

處理完現有資料後，再重新啟動程式，讓 JPA 在 FK 欄位建立約束（constraint）即可。往後新增的學生資料，也都必須能關聯到一個已存在的科系。

## 三、儲存一對多關聯的資料
為了確認一對多關聯的效果，我們會在 Controller 設計 RESTful API，透過 repository 存取資料庫。

### （一）Request 與 Response body
以下是用來建立學生的 request body。包含名字、科系編號，以及聯繫方式。
``` java
public class StudentRequest {
    private String name;
    private Long departmentId;
    private Contact contact;
    
    // getter, setter ...
}
```

在 request body 攜帶 Contact 物件，是為了在程式中直接賦予給 Student 實體物件，再透過級聯的機制，一起插入到 table。

而科系並非攜帶整個物件，而是編號。理由是科系資料應該早就在 table 準備好了，等著學生資料的 FK 來指向。

以下是學生資料的 response body，包含 id、名字、科系名稱、信箱與電話，共 5 個欄位。
``` java
public class StudentResponse {
    private Long id;
    private String name;
    private String departmentName;
    private String email;
    private String phone;
    
    // getter, setter ...
}
```

### （二）關聯插入
以下的 API 是用來建立學生。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;

    @Autowired
    private DepartmentRepository departmentRepository;
    
    @PostMapping("/students")
    public ResponseEntity<Void> createStudent(@RequestBody StudentRequest request) {
        Optional<Department> departmentOp = departmentRepository.findById(request.getDepartmentId());
        if (departmentOp.isEmpty()) {
            return ResponseEntity.unprocessableEntity().build();
        }

        Student student = new Student();
        student.setName(request.getName());
        student.setContact(request.getContact());
        student.setDepartment(departmentOp.get());
        studentRepository.save(student);

        return ResponseEntity.noContent().build();
    }
}
```

邏輯中，會先檢查該科系編號是否存在。是的話，便取出 request body 中的資料，建立出 Student 物件。

接著讓 Student 物件攜帶科系資料的實體，並呼叫 repository 的 `save` 方法來儲存。此時 JPA 會自動在學生資料的 FK，也就是 dept_id 欄位，填入科系編號。

### （三）關聯查詢
以下的 API，用途是透過學生名字的關鍵字來查詢。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;

    @Autowired
    private DepartmentRepository departmentRepository;
    
    @GetMapping("/students")
    public ResponseEntity<List<StudentResponse>> getStudents(
            @RequestParam(required = false, defaultValue = "") String name
    ) {
        List<Student> students = studentRepository.findByNameLikeIgnoreCase("%" + name + "%");
        Map<Student, Department> studentDepartmentMap = createStudentDepartmentMap(students);
        Map<Student, Contact> studentContactMap = createStudentContactMap(students);

        List<StudentResponse> responses = students
                .stream()
                .map(s -> {
                    Department department = studentDepartmentMap.get(s);
                    Contact contact = studentContactMap.get(s);
                    return StudentResponse.of(s, contact, department);
                })
                .toList();

        return ResponseEntity.ok(responses);
    }
    
    private Map<Student, Department> createStudentDepartmentMap(Collection<Student> students) {
        Map<Student, Long> studentDepartmentIdMap = students
                .stream()
                .collect(Collectors.toMap(Function.identity(), s -> s.getDepartment().getId()));

        List<Department> departments = departmentRepository.findAllById(studentDepartmentIdMap.values());
        Map<Long, Department> departmentMap = departments
                .stream()
                .collect(Collectors.toMap(Department::getId, Function.identity()));

        Map<Student, Department> map = new HashMap<>();
        students.forEach(s -> {
            Long deptId = studentDepartmentIdMap.get(s);
            Department dept = departmentMap.get(deptId);
            map.put(s, dept);
        });

        return map;
    }

    private Map<Student, Contact> createStudentContactMap(Collection<Student> students) {
        // ...
    }
}
```

首先呼叫 repository 查詢學生資料。由於 Department 欄位的加載策略設為 `FetchType.LAZY`，因此不會立刻查詢科系資料。

接著仿照上一篇的做法，為了避免 N + 1 問題，因此將科系編號收集起來，透過 repository 一次查詢出所有科系。最後把結果組裝成 response body。

### （四）托管狀態
在上面建立學生資料的例子中，有件事要留意。儲存 Student 物件時，它所關聯的 Department 物件，必須是從 repository 查詢出的實體。

原因是從 repository 查詢出的實體，會被 JPA 的「實體管理器」（Entity Manager）所管理，我們稱它處於「托管狀態」（managed）。

只有攜帶托管狀態的實體一起儲存時，Entity Manager 才會知道要將這些資料關聯在一起。這也是為什麼在儲存 Student 物件時，JPA 可以自動處理關聯，在 FK 欄位填入科系編號的值。

如果我們自行建立要關聯的 Department 物件（即便內容與 table 現有的相同），並賦予給 Student 物件後儲存，那麼該 Department 物件是處於非托管狀態（detached）的。由於 Entity Manager 不認識它，因此會視為新資料，在科系表插入，形成重複資料。

## 四、雙向關聯
在目前的設計中，查詢到 Student 實體後，可以透過呼叫「getDepartment()」方法來取得科系的資料。

那麼換一個角度，如果在程式邏輯中已經查詢到 Department 實體了，當我們想取得就讀該科系的所有學生，該如何做呢？其中一種做法，是在 StudentRepository 宣告查詢方法，直接以 FK 欄位做為條件。
``` java
@Repository
public interface StudentRepository extends JpaRepository<Student, Long> {

    @Query(
            nativeQuery = true,
            value = "SELECT * FROM `student` WHERE `dept_id` = ?1"
    )
    List<Student> findByDepartmentId(Long id);
}
```

但這麼做的話，程式就勢必依賴於 StudentRepository 這個元件。一般來說，我們會希望元件的耦合度低，也就是不要依賴太多元件。

透過配置「雙向關聯」，可以讓我們在任何一方的實體，皆能獲取所關聯的另一方實體。以本文的例子來看，意思就是在查詢到 Department 實體後，也能直接取得所有關聯的 Student 實體。
``` java
@Entity
@Table(name = "department")
public class Department {
    // ...

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "department")
    private Set<Student> students;

    // getter, setter ...
}
```

以上在 Department 類別添加了 `Set<Student>` 的欄位，並冠上 `@OneToMany` 注解，表明一對多關聯。該注解的 `mappedBy` 參數傳入了欄位名稱，是用來宣告這個實體，是被另一方實體的哪個欄位所關聯。

也就是說，在 Department 類別的欄位使用 `mappedBy` 參數，等於宣告 Student 類別的 FK 位於叫做「department」的欄位上。如此便完成了雙向關聯的配置。

附帶一提，使用 Set 資料結構來攜帶 Student 實體，用意是強調查詢結果沒有順序之分。

配置完後，讓我們看看效果。以下的 API，是取得指定科系的學生。
``` java
@RestController
public class MyController {
    // ...

    @Autowired
    private DepartmentRepository departmentRepository;
    
    @GetMapping("/departments/{id}/students")
    public ResponseEntity<List<StudentResponse>> getStudentsByDepartment(@PathVariable Long id) {
        Optional<Department> departmentOp = departmentRepository.findById(id);
        if (departmentOp.isEmpty()) {
            return ResponseEntity.notFound().build();
        }

        Department department = departmentOp.get();
        Set<Student> students = department.getStudents();
        List<StudentResponse> responses = students
                .stream()
                .map(s -> {
                    StudentResponse res = new StudentResponse();
                    res.setId(s.getId());
                    res.setName(s.getName());

                    return res;
                })
                .toList();

        return ResponseEntity.ok(responses);
    }
}
```

首先呼叫 repository 查詢科系。接著呼叫「getStudents()」方法的同時，JPA 便會進行查詢學生表。在 console 印出的 SQL 指令，示意如下：
``` text
Hibernate:
    SELECT s.id, s.name, s.dept_id
    FROM student s
    WHERE s.dept_id = ?
```

由於 `mappedBy` 參數指定了 Student 的 department 欄位，且冠上的 `@JoinColumn` 注解已定義 FK 的欄位名稱為「dept_id」，因此 JPA 才知道要以什麼條件來查詢 Student。

要注意的是，透過雙向關聯取得關聯資料時，無法進行條件篩選、排序與分頁。不過若讀者沒有這項需求，配置雙向關聯依然是相當實用的做法。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.5-jpa-one-to-many-relationship-and-bidirectional-association)。

上一課：<a href="/articles/spring-boot-jpa-one-to-one-relationship" target="_blank">【Spring Boot】第9.4課－使用 JPA 配置資料表關聯（以一對一關聯為例）</a>

下一課：<a href="/articles/spring-boot-jpa-many-to-many-relationship-and-intermediary-table" target="_blank">【Spring Boot】第9.6課－使用 JPA 建立多對多關聯，並配置中間表</a>