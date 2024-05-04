# 엔티티 그래프

> 항목 39, 41에서 Fetch Join이 있음
> 
- = fetch plans
- 지연 로딩 예외와 N+1 문제를 해결할 수 있음
- 엔티티와 관련된 연관관계와, SELECT 문에 로드돼야 할 필드를 지정
- 엔티티에 대한 여러 엔티티 그래프를 정의해 다른 엔티티와 연결
- sub-graph를 사용해 복잡한 fetch plan을 정의
- 여러 엔티티에서 재사용 될 수 있음

## 예시

- `Author - Book` : 양방향 엔티티 그래프

```java
@Entity
@NamedEntityGraph(
    name = "author-books-graph",
    attributeNodes = {
        @NamedAttributeNode("books")
    })
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String genre;
    private int age;

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author", orphanRemoval = true)
    private List<Book> books = new ArrayList<>();
}

@Entity
@NamedEntityGraph(
    name = "books-author-graph",
    attributeNodes = {
        @NamedAttributeNode("author")
    })
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String isbn;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;
}
```

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {
    @Override
    @EntityGraph(value = "author-books-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findAll();

    @EntityGraph(value = "author-books-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findByAgeLessThanOrderByNameDesc(int age);

    @EntityGraph(value = "author-books-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    @Query(value="SELECT a FROM Author a WHERE a.age > 20 AND a.age < 40")
    List<Author> fetchAllAgeBetween20And40();
}
```

## 엔티티 서브 그래프 예시

- Book 테이블은 다수의 Author와 Publisher를 가진다.
- Author를 조회하며, Author와 관련된 Book과 그 Book과 관련된 Publisher를 가져오고 싶다.
- 엔티티 서브그래프를 이용해 구현

```java
@Entity
@NamedEntityGraph(
    name = "author-books-publisher-graph",
    attributeNodes = {
        @NamedAttributeNode(value = "books", subgraph = "publisher-subgraph")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "publisher-subgraph",
            attributeNodes = {
                @NamedAttributeNode("publisher")
            }
        )
    }
)
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    ...

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author", orphanRemoval = true)
    private List<Book> books = new ArrayList<>();
}

@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    ...

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "publisher_id")
    private Publisher publisher;
}

@Entity
public class Publisher {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String company;

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "publisher", orphanRemoval = true)
    private List<Book> books = new ArrayList<>();
}
```

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {
    @Override
    @EntityGraph(value = "author-books-publisher-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findAll();

    @EntityGraph(value = "author-books-publisher-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findByAgeLessThanOrderByNameDesc(int age);

    @EntityGraph(value = "author-books-publisher-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    @Query(value="SELECT a FROM Author a WHERE a.age > 20 AND a.age<40")
    List<Author> fetchAllAgeBetween20And40();
}
```

- 조회 쿼리를 보면 Left Outer Join 으로 가져온다.

## 부분만 가져오기

- 속성값
    - fetch graph: 지정 값은 Eager로 가져오고, 나머지는 무조건 Lazy로 가져옴
    - load graph: 지정 값은 Eager로 가져오고, 나머지는 기본 FetchType 으로 가져옴

## Book-Author 예시

```java
@Entity
@NamedEntityGraph(
    name = "author-books-graph",
    attributeNodes = {
        @NamedAttributeNode("name"),
        @NamedAttributeNode("books")
    }
)
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    @Basic(fetch = FetchType.LAZY)
    private String genre;
    @Basic(fetch = FetchType.LAZY)
    private int age;

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author", orphanRemoval = true)
    private List<Book> books = new ArrayList<>();
}
```

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {
    @EntityGraph(value = "author-books-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findByAgeGreaterThanAndGenre(int age, String genre);
}
```

- EntityGraph 속성으로 name과 book만 지정했다. 그럼 나머지 genre와 age는 안가져올까?
    - no, JPA 기본 속성이 EAGER이기 때문에 가져온다.
- 가져오지 않게 하기 위해서는 설정을 변경해야 한다.
    - 코드를 아래와 같이 변경하고, Maven에 orm.tooling 플러그인을 추가해줘야 한다. (p.123)

```java
public class Author {
    ...
    private String name;
    @Basic(fetch = FetchType.LAZY)  // 추가
    private String genre;
    @Basic(fetch = FetchType.LAZY)  // 추가 -> 기본 설정 변경
    private int age;
    ...
}
```

- 설정 후 쿼리문에서는 genre와 age를 가져오지 않는다.

# `@Where` 어노테이션

```java
@Entity
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String genre;
    private int age;

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author", orphanRemoval = true)
    private List<Book> books = new ArrayList<>();

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author", orphanRemoval = true)
    @Where(clause = "price <= 20")
    private List<Book> cheapBooks = new ArrayList<>();

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author", orphanRemoval = true)
    @Where(clause = "price > 20")
    private List<Book> restOfBooks = new ArrayList<>();
}
```

- 기본적인 where 쿼리를 붙혀준다.
- soft delete 컬럼 등에 사용할만 함
