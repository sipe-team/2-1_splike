# 양방향 `@OneToMany`

- 양방향 관계는 항상 ‘부모 → 자식’ 방향으로 전이되도록 설정

```java
// 부모 객체
@Entity
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author",         // 양방향 의존관계 설정, 관계의 주인은 author
        orphanRemoval = true)        // 고아객체 자동 삭제
    private List<Book> books = new ArrayList<>();
    
    // 자식 관리용 메소드
    public void addBook(Book book) {
        this.books.add(book);
        book.setAuthor(this);
    }
    public void removeBook(Book book) {
        book.setAuthor(null);
        this.books.remove(book);
    }
    public void removeBooks() {
        Iterator<Book> iterator = this.books.iterator();
        while (iterator.hasNext()) {
            Book book = iterator.next();
            book.setAuthor(null);
            iterator.remove();
        }
    }
}

// 자식 객체
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @ManyToOne(fetch = FetchType.LAZY)  // 지연로딩
    @JoinColumn(name = "author_id")     // 명시적으로 필드명 설정
    private Author author;
}
```

# 단방향 `@OneToMany` 의 문제점

## `@OneToMany` 만 사용

```java
// 부모 객체
@Entity
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    // @OrderColumn(name = "books_order")
    // @JoinColumn(name = "author_id")
    private List<Book> books = new ArrayList<>();

    // 자식 관리용 메소드
    public void addBook(Book book) {
        this.books.add(book);
        // book.setAuthor(this);
    }
    public void removeBook(Book book) {
        // book.setAuthor(null);
        this.books.remove(book);
    }
    public void removeBooks() {
        Iterator<Book> iterator = this.books.iterator();
        while (iterator.hasNext()) {
            Book book = iterator.next();
            // book.setAuthor(null);
            iterator.remove();
        }
    }
}

// 자식 객체
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    
    // @ManyToOne(fetch = FetchType.LAZY)  // 지연로딩
    // @JoinColumn(name = "author_id")     // 명시적으로 필드명 설정
    // private Author author;
}
```

### 문제점

1. 별도의 연결테이블 생성
    - 단방향 OneToMany는 Book, Author에 FK가 생기지 않고, 별도의 연결 테이블에 FK 두 개가 정의되어 추가됨. ex) author_book 관계 테이블 생성
        
        → 관리 포인트가 많아져 신경쓸 부분도 많아지고, 성능도 떨어짐
        
2.  기존 Author에 새로운 Book 등록 시 비효율
    - 관계 테이블에서 해당 author와 관련된 값을 전부 삭제하고 다시 등록
3. 첫번째, 마지막 Book 삭제 시 비효율
    - 관계 테이블에서 해당 author와 관련된 값을 전부 삭제하고 다시 등록

## `@OrderColumn` 추가

```java
// 부모 객체
@Entity
public class Author {
    ...
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @OrderColumn(name = "books_order")
    private List<Book> books = new ArrayList<>();
```

### 문제점 개선

1. 인덱스를 이용해 약간의 개선이 있지만 여전히 비효율
    - 마지막 Book 삭제 시 해당 Book과 관계 테이블 row만 삭제
    - 첫번째 Book 삭제 시 해당 Book과 관계 테이블 row만 삭제, 하지만 순서를 맞추기 위해 나머지 관계 테이블 전체 row에 Update 쿼리

## `@JoinColumn` 추가

```java
// 부모 객체
@Entity
public class Author {
    ...
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "author_id")
    private List<Book> books = new ArrayList<>();
```

- 추가하면 FK가 book에 생성되며, 별도의 관계 테이블이 생성되지 않는다.

### 문제점

1. 새로 추가
    - author, book을 각각 INSERT 후, UPDATE book으로 FK 연결해줌 → 비효율
2. 첫번째, 마지막 Book 삭제
    - UPDATE book으로 FK 연결 해제 후 DELETE book 진행 → 비효율

# 노트

- 영속성 컨텍스트
    - 앱과 DB 사이에 객체를 저장하는 가상의 DB 역할 수행
    - 엔티티 메니저를 통해 객체를 처리할 때 사용
    - 내부에 캐시를 갖는데 이를 1차 캐시라 함
    - JPA 구현체에서 별도로 애플리케이션 수준의 캐시를 지원하는데 이를 2차 캐시 또는 공유 캐시라 함

- 신기했던 부분
    - 처음보는 어노테이션인 `@OrderColmn`은 인덱스와 관련이 있을까 싶었는데 아니었음. 그저 일대다, 다대다에서 다측의 컬렉션을 정렬해서 저장하고자 할 때 사용하는 것. 상황에 따라 인덱스로 지정하여 활용하기도 한다고 함.