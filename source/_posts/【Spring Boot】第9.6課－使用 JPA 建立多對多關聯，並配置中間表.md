---
title: 【Spring Boot】第9.6課－使用 JPA 建立多對多關聯，並配置中間表
date: 2024-06-12 11:30:00
permalink: articles/spring-boot-jpa-many-to-many-relationship-and-intermediary-table
index_img: /img/index_img/spring-boot.jpg
excerpt: 本文將進一步以「學生」選修「課程」為情境，建立多對多關聯，並撰寫 RESTful API 進行測試。過程中也會介紹多對多關聯特有的「中間表」（intermediary table），並說明如何在程式中正確地操作實體，讓 JPA 維護雙方的關聯。
categories:
  - ["Spring Boot"]
---

在上一篇，已經示範過如何建立一對多關聯。本文將進一步以「學生」選修「課程」為情境，建立多對多關聯，並撰寫 RESTful API 進行測試。

過程中也會介紹多對多關聯特有的「中間表」，並說明如何正確地操作，才能讓 JPA 在中間表維護雙方的關聯。最後配置雙向關聯，站在不同的角度，查詢另一方的資料。


-----


本文的練習用專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.5-jpa-one-to-many-relationship-and-bidirectional-association)。

## 一、程式專案準備
### （一）實體類別介紹
以下的「Student」類別，描述了學生資料，包含 id、名字、聯繫方式與科系，共 4 個欄位。其中聯繫方式與科系是來自前面文章的示範，本文盡量不提及。
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

    @ManyToOne(fetch = FetchType.LAZY, cascade = {CascadeType.PERSIST})
    @JoinColumn(name = "dept_id", referencedColumnName = "id", nullable = false)
    private Department department;
    
    // getter, setter ...
}
```

以下的「Course」類別，描述了課程，包含 id 與名稱這 2 個欄位。
``` java
@Entity
@Table(name = "course")
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // getter, setter ...
}
```

由於 Course 也是要儲存到資料庫的實體類別，因此需準備 repository。
``` java
@Repository
public interface CourseRepository extends JpaRepository<Course, Long> {
}
```

### （二）準備測試資料
定義好實體類別後，讀者可啟動程式，讓 Spring Data JPA 建立 table。

若有需要，可執行以下 SQL 指令，建立課程的測試資料。畢竟使用情境中，必定是先有課程，才會有學生來選修。
``` sql
INSERT INTO `course` (`name`)
VALUES ("資料庫管理"), ("投資學"), ("審計學");
```

## 二、設計多對多關聯
### （一）設計方式
首先複習一下先前文章介紹的關聯。
* 一對一關聯：每位學生擁有一組聯繫方式，每組聯繫方式只有一位擁有者。
* 一對多關聯：每個科系有多位學生就讀，每位學生只能就讀一個科系。

在 table 的設計上，會添加額外的外鍵（Foreign Key，FK）欄位，關聯到另一張 table 的主鍵（Primary Key，PK）。

本文的學生與課程，屬於多對多關聯。在情境上，每位學生可選修多個課程，而每個課程也可被多位學生選修。然而在 table 設計上，一個 FK 欄位是無法儲存多個 PK 的。

為了建立多對多關聯，我們需要另外建立一張「中間表」（intermediary table），記錄什麼學生選修了什麼課程。示意如下：
|學生|課程|
|-|-|
|Vincent|資料庫管理|
|Vincent|投資學|
|Ivy|投資學|
|Ivy|審計學|

而下圖是 table 的設計，讀者可看出除了學生表與課程表，還準備了一個叫做「student_course」的中間表。
<img src="{{ permalink }}many-to-many-relationship-student-course.png" />

從圖中可看到，中間表的欄位有學生與課程編號。這 2 個欄位會一起做為 PK 欄位（複合主鍵），確保該組合不重複，意即學生不可重複選修相同的課程。

中間表的用途是記錄兩張 table 的資料對應關係，欄位均作為 FK，分別關聯到雙方 table 的 PK。也就是說，多對多關聯是讓 2 張 table 對中間表建立一對多關聯來達成。

### （二）程式配置
回到程式專案，以下是在 Student 實體類別配置關聯。
``` java
@Entity
@Table(name = "student")
public class Student {
    // ...

    @ManyToMany(fetch = FetchType.LAZY, cascade = {CascadeType.PERSIST})
    @JoinTable(
            name = "student_course",
            joinColumns = @JoinColumn(name = "student_id", referencedColumnName = "id"),
            inverseJoinColumns = @JoinColumn(name = "course_id", referencedColumnName = "id"),
            uniqueConstraints = @UniqueConstraint(columnNames = {"student_id", "course_id"})
    )
    private Set<Course> courses;

    // getter, setter ...
}
```

此處在實體類別添加了 `Set<Course>` 欄位，並冠上 `@ManyToMany` 注解，表明多對多關聯。接著透過 `@JoinTable` 注解來配置中間表。

配置中間表時，會用到多個參數。`name` 參數是定義中間表的名稱；`joinColumns` 參數是站在中間表的角度，定義第一個 FK 欄位，取名為「student_id」。並提供關聯到實體類別（Student）的 PK 欄位名稱，也就是 id。

而 `inverseJoinColumns` 參數，則在中間表定義了第二個 FK 欄位，取名為「course_id」。它會關聯到另一個實體類別（Course）的 PK 欄位。

最後是 `uniqueConstraints` 參數，其用途是將這 2 個欄位設為 PK，目的是確保不會插入相同組合的值。

## 三、儲存多對多關聯的資料
為了確認多對多關聯的效果，我們會在 Controller 設計 RESTful API，透過 repository 存取資料庫。

### （一）Request 與 Response body
以下是進行課程選修的 request body，可攜帶多個課程編號。
``` java
public class TakeCourseRequest {
    private Set<Long> courseIds;

    // getter, setter ...
}
```

以下是課程資料的 response body，包含了 id 與名稱這 2 個欄位。
``` java
public class CourseResponse {
    private Long id;
    private String name;
    
    // getter, setter ...
}
```

以下是學生資料的 response body，包含了 id 與名稱這 2 個欄位。
``` java
public class StudentResponse {
    private Long id;
    private String name;

    // getter, setter ...
}
```

### （二）關聯插入
以下的 API 是為某位學生進行課程選修。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;    
    
    @Autowired
    private CourseRepository courseRepository;

    @PostMapping("/students/{id}/courses")
    public ResponseEntity<Void> takeCourse(
            @PathVariable Long id, @RequestBody TakeCourseRequest request
    ) {
        Optional<Student> studentOp = studentRepository.findById(id);
        if (studentOp.isEmpty()) {
            return ResponseEntity.notFound().build();
        }

        List<Course> targetCourses = courseRepository.findAllById(request.getCourseIds());

        Student student = studentOp.get();
        Set<Course> existingCourses = student.getCourses();
        existingCourses.addAll(targetCourses);
        studentRepository.save(student);

        return ResponseEntity.noContent().build();
    }
}
```

首先將學生及 request body 所指定的課程都從 repository 查詢出來。另外，呼叫 Student 實體的「getCourses()」方法，本身也會在資料庫查詢該學生原本已選修的課程。

上一篇有提到，從資料庫直接查詢出來的資料，都處於「托管狀態」，被實體管理器（Entity Manager）所管理。因此不論是從 Student 實體查出的課程，還是從 repository 查出的課程，若有重複，則它們在記憶體中皆為同一個 instance。

在 Student 實體類別中，是以 Set 資料結構來攜帶多個 Course 實體。根據其特性，即便新增原本就存在的課程資料進去，也不會造成重複（畢竟記憶體位置相同）。

綜上所述，JPA 會在將 Student 實體存回 repository 時，在中間表處理資料的關聯。並且只針對那些新資料，在中間表進行插入。

呼叫 API 後，JPA 在 console 印出的部份 SQL 指令，示意如下：
``` text
Hibernate:
    INSERT INTO student_course (student_id, course_id)
    VALUES (?, ?)
Hibernate:
    INSERT INTO student_course (student_id, course_id)
    VALUES (?, ?)
```

若有 2 筆新的 Course 被添加到 Student 中，那麼就會在中間表插入 2 筆資料。

### （三）關聯刪除
以下的 API 是讓學生退選某個課程。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;
    
    @Autowired
    private CourseRepository courseRepository;
    
    @DeleteMapping("/students/{studentId}/courses/{courseId}")
    public ResponseEntity<List<CourseResponse>> deleteStudentTakingCourses(
            @PathVariable Long studentId, @PathVariable Long courseId
    ) {
        Optional<Student> studentOp = studentRepository.findById(studentId);
        if (studentOp.isEmpty()) {
            return ResponseEntity.notFound().build();
        }

        Student student = studentOp.get();
        student.getCourses().removeIf(c -> c.getId().equals(courseId));
        studentRepository.save(student);

        return ResponseEntity.noContent().build();
    }
}
```

此處是先將 Student 實體查詢出來，接著從它持有的 Course 集合中，移除指定 id 的資料。最後存回 repository 時，JPA 會針對 Course 集合所缺少的資料，從中間表刪除。

呼叫 API 後，JPA 在 console 印出的部份 SQL 指令，示意如下：
``` text
Hibernate:
    DELETE FROM student_course
    WHERE student_id = ? AND course_id = ?
```

讀者可看到，不論是插入還是刪除，JPA 都會根據資料的增減，對中間表進行維護。

### （四）關聯查詢
以下的 API，是查詢某位學生所選修的課程。
``` java
@RestController
public class MyController {

    @Autowired
    private StudentRepository studentRepository;
    
    @GetMapping("/students/{id}/courses")
    public ResponseEntity<List<CourseResponse>> getStudentTakingCourses(@PathVariable Long id) {
        Optional<Student> studentOp = studentRepository.findById(id);
        if (studentOp.isEmpty()) {
            return ResponseEntity.notFound().build();
        }

        Student student = studentOp.get();
        List<CourseResponse> courses = student.getCourses()
                .stream()
                .map(c -> {
                    CourseResponse res = new CourseResponse();
                    res.setId(c.getId());
                    res.setName(c.getName());
                    return res;
                })
                .toList();

        return ResponseEntity.ok(courses);
    }
}
```

邏輯很單純，查詢出 Student 實體後，再從中查詢所關聯的多個 Course，將它們包裝為 response body 即可。

呼叫 API 後，JPA 在 console 印出的部份 SQL 指令，示意如下：
``` text
Hibernate:
    SELECT sc.student_id, c.id, c.name 
    FROM student_course sc
    JOIN course c ON c.id = sc.course_id
    WHERE sc.student_id = ?
```

可看到查詢選修課程的方式，是將中間表與課程表做關聯，再以學生編號做為查詢條件。

## 四、雙向關聯
在多對多關聯中，同樣也能使用雙向關聯。以下是在 Course 實體類別進行配置，藉此取得所關聯的 Student 實體。
``` java
@Entity
@Table(name = "course")
public class Course {
    // ...

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students;

    // getter, setter ...
}
```

此處添加了 `Set<Student>` 的欄位，並冠上 `@ManyToMany` 注解，表明了多對多關聯。該注解的 `mappedBy` 參數傳入了欄位名稱，是用來宣告這個 Course 實體，是被 Student 實體類別的哪個欄位所關聯。

以下的 API，是查詢某個課程的選修學生。
``` java
@RestController
public class MyController {

    @Autowired
    private CourseRepository courseRepository;
    
    @GetMapping("/courses/{id}/students")
    public ResponseEntity<List<StudentResponse>> getCourseTakingStudents(@PathVariable Long id) {
        Optional<Course> courseOp = courseRepository.findById(id);
        if (courseOp.isEmpty()) {
            return ResponseEntity.notFound().build();
        }

        Set<Student> students = courseOp.get().getStudents();
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

邏輯很單純，查詢出 Course 實體後，再從中查詢所關聯的多個 Student，將它們其包裝為 response body 即可。

呼叫 API 後，JPA 在 console 印出的部份 SQL 指令，示意如下：
``` text
Hibernate:
    SELECT sc.course_id, s.id, s.name
    FROM student_course sc
    JOIN student s ON s.id = sc.student_id
    WHERE sc.course_id = ?
```

可看到查詢修課學生的方式，是將中間表與學生表做關聯，再以課程編號做為查詢條件。


-----


本文的完成後專案，請[點我](https://github.com/ntub46010/SpringBootTutorial/tree/Ch09.6-jpa-many-to-many-relationship-and-intermediary-table)。

上一課：<a href="/articles/spring-boot-jpa-one-to-many-relationship-and-bidirectional-association" target="_blank">【Spring Boot】第9.5課－使用 JPA 建立一對多關聯，並配置雙向關聯</a>

下一課：<a href="/articles/spring-boot-mockmvc-integration-test" target="_blank">【Spring Boot】第10.1課 - 使用 MockMvc 進行 API 整合測試</a>