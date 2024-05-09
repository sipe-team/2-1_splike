# 항목 56 `@ElementCollection` 컬렉션 JOIN FETCH 방법

단방향 일대다 연관관계를 정의할 때 JPA는 `@ElementCollection`라는 간단한 방법 제공

`@CollectionTable`을 통해 커스텀될 수 있는 별도 테이블로 매핑된다.

```java
@Entity
public class ShoppingCart implements Serializable {

  // ...
  @ElementCollection(fetch = FetchType.LAZY)
  @CollectionTable(name = "shopping_cart_books",
          joinColumns = @JoinColumn(name = "shopping_cart_id"))
  private List<Book> books = new ArrayList<>();
}
```

기본적으로 도서는 지연 로딩이며, 즉시 도서를 가져오고 싶다면 JOIN FETCH 를 사용한다.

```java
@Repository
public interface ShoppingCartRepository
       extends JpaRepository<ShoppingCart, Long> {

  @Query(value = "SELECT p FROM ShoppingCart p JOIN FETCH p.books")
  ShoppingCart fetchShoppingCart();

  @Query(value = "SELECT p FROM ShoppingCart p
                  JOIN FETCH p.books b WHERE b.price > ?1")
  ShoppingCart fetchShoppingCartByPrice(int price);
}
```

# 항목 59: 엔터티 컬렉션 병합 방법

엔티티 컬렉션 병합

Author와 Book이 양방향 지연 @OneToMany 연관관계를 갖고 있다고 가정

```java
@Entity
public class Author implements Serializable {

  private static final long serialVersionUID = 1L;

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;
  private String genre;
  private int age;

  @OneToMany(cascade = CascadeType.ALL,
             mappedBy = "author", orphanRemoval = true)
  private List<Book> books = new ArrayList<>();

  public void addBook(Book book) {
    this.books.add(book);
    book.setAuthor(this);
  }

  public void removeBook(Book book) {
    book.setAuthor(null);
    this.books.remove(book);
  }

  public void removeBooks() {
    // ...
  }
}
```

```java
@Entity
public class Book implements Serializable {

  private static final long serialVersionUID = 1L;

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String title;
  private String isbn;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "author_id")
  private Author author;

  @Override
  public boolean equals(Object obj) {
    if (obj == null) {
      return false;
    }
    if (this == obj) {
      return true;
    }
    if (getClass() != obj.getClass()) {
      return false;
    }

    return id != null && id.equals(((Book) obj).id);
  }

  @Override
  public int hashCode() {
    return 2021;
  }

  @Override
  public String toString() {
    return "Book{" + "id=" + id + ", title=" + title + ", \
           isbn=" + isbn + '}';
  }
}
```

Author 레코드와 관련있는 Book 엔터티 목록 가져오기 위해서 JOIN 사용

```java
@Repository
public interface BookRepository extends JpaRepository<Book, Long> {

  @Query(value = "SELECT b FROM Book b JOIN b.author a WHERE a.name = ?1")
  List<Book> booksOfAuthor(String name);
}
```

반환된 `List<Book>`은 detached 되었고, 수정이 자동으로 데이터베이스에 반영되지 않음

## Detached 컬렉션 병합

최소 데이터베이스 처리 횟수를 사용해 분리된 컬렉션을 병합한다.

```java
@Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {
  @Query(value = "SELECT a FROM Author a JOIN FETCH a.books WHERE a.name = ?1")
  Author authorAndBooks(String name);
}
```

JOIN FETCH를 사용하면 된다.

수동 병합 3단계

- 추가되는 컬렉션에서 더 이상 존재하지 않는 기존 데이터베이스 행을 제거한다. 첫째, detachedBooks에 없는 author의 도서를 필터링한다. 둘째, detachedBooks에서 찾을 수 없는 모든 author 도서는 제거해야 한다.

```java
List<Book> booksToRemove = author.getBooks().stream()
  .filter(b -> !detachedBooks.contains(b))
  .collect(Collectors.toList());

booksToRemove.forEach(b -> author.removeBook(b));
```

- 추가되는 컬렉션에 존재하는 기존 데이터베이스 행을 업데이트한다, 첫째, 새 도서를 필터링한다. 이 도서들은 detachedBooks에 있지만 author의 도서에는 없는 것들이다. 둘째, detachedBooks를 필터링해 detachedBooks에는 있지만 newBooks에 없는 도서를 가져온다.

```java
List<Book> newBooks = detachedBooks.stream()
  .filter(b -> !author.getBooks().contains(b))
  .collect(Collectors.toList());

detachedBooks.stream()
  .filter(b -> !newBooks.contains(b))
  .forEach((b) -> {
    b.setAuthor(author);
    Book mergedBook = bookRepository.save(b);
    author.getBooks().set(
      author.getBooks().indexOf(mergedBook), mergedBook);
  });
```

- 마지막으로 현재 결과 세트에 존재하지 않는 추가되는 컬렉션의 행들을 등록

```java
newBooks.forEach(b -> author.addBook(b));
```

# 항목 103: 하이버네이트 HINT_PASS_DISTINCT_THROUGH를 통한 SELECT DISTINCT 최적화 방법

- 양방향 지연 일대다 연관관계를 갖는 Author와 Book 엔터티를 고려.

모든 Book 자식 엔터티와 함께 Author 엔터티 목록을 가져온다고 하면, 한 저자의 책 데이터를 모두 가져오는 것은 SQL 단에서 book 테이블에서 가져온 행의 개수와 같다.
(ex. 한 저자가 두 개의 책을 썼고, Author와 함께 Book을 모두 가져오면 Book은 총 두 개이므로 Book의 행 개수와 같다.)
-> Author 중복이 발생할 수 있다.

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Query("SELECT a FROM Author a LEFT JOIN FETCH a.books")
  List<Author> fetchWithDuplicates();
}
```

만약 20권의 도서를 저술한 다작저자라면, 동일한 Author 엔터티에 대한 20개의 참조를 갖는 것은 원치 않은 성능 저하가 된다.

**DISTINCT** 키워드를 사용하여 해결할 수 있다.

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books")
  List<Author> fetchWithoutHint();
}
```

`List<Author>`에서 중복 항목이 제거됐지만 DISTINCT 키워드가 데이터베이스로 전달되어서 유니크한 부모-자식 레코드가 포함되어 있어도 PostgreSQL, MySQL 실행계획에 DISTINCT가 수반되며 중복 제거를 위한 오버헤드가 발생하게 된다.

하이버네이트 5.2.2의 `QueryHints.HINT_PASS_DISTINCT_THROUGH`로 처리할 수 있다.

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Query("SELECT DISTINCT a FROM Author a LEFT JOIN FETCH a.books")
  @QueryHints(value = @QueryHint(name = HINT_PASS_DISTINCT_THROUGH,
              value = "false"))
  List<Author> fetchWithHint();
}
```

SQL 쿼리에 DISTINCT가 사라지며, 중복되는 Author 데이터도 사라진다.
또한, 실행 계획에도 더 이상 불필요한 오버헤드가 포함되지 않는다.

# 항목 105: 스프링 데이터 쿼리 빌더를 통한 결과 세트 크기 제한과 카운트 및 삭제 파생 쿼리 사용 방법

제목부터 어렵다...

스프링 데이터는 JPA를 위한 쿼리 빌더 메커니즘을 제공하며 쿼리 메서드 이름을 해석해 이를 SQL문으로 변환한다.

## 결과 세트 크기 제한

실제 사용하는 데이터에 비해 오버하게 데이터를 가져오면 안 된다 -> 페이지네이션 같이 결과 세트를 분리하는 게 좋음.

### LIMIT

기본적으로 쿼리 메서드 이름을 기재하여 LIMIT를 추가할 수 있다. first 또는 top으로 제한할 수 있다.

56세인 첫 5명의 저자를 가져온다고 가정해보자.

```java
List<Author> findTop5ByAge(int age);
```

또는 first 키워드를 사용한다.

```java
List<Author> findFirst5ByAge(int age);
```

내부적으로 SQL로 변환된다.

### SORT

결과 세트를 정렬해야 한다면, `OrderByPropertyDesc/Asc`를 사용한다.

```java
List<Author> findFirst5ByAgeOrderByNameNameDesc(int age);
```

50세 미만인 Horror 장르의 첫 5명의 저자를 name 내림차순으로 가져온다면 LessThan 키워드를 추가한다.

```java
List<Author> findFirst5ByGenreAndAgeLessThanOrderByNameDesc(
  String genre, int age);
```

### Page, Slice

```java
Page<Author> queryFirst10ByName(String name, Pageable p)
Slice<Author> findFirst10ByName(String name, Pageable p)
```

### 스프링 프로젝션(DTO)와 함께 사용 가능

```java
public interface AuthorDto {
  String getName();
  String getAge();
}
```

다음과 같이 결과 세트를 가져올 수 있다.

```java
List<AuthorDto> findFirst5ByOrderByAgeAsc();
```

## 카운트 및 삭제 파생 쿼리

쿼리 빌더 메커니즘에는 find...By 타입의 쿼리 외에도 파생된 카운트 쿼리와 삭제 쿼리를 제공

### 파생 카운트 쿼리

count...By로 시작

```java
long countByGenre(String genre);
```

### 파생 삭제 쿼리

삭제된 레코드 수 또는 목록을 반환할 수 있다. delete...By 또는 remove...By로 시작하고 long을 반환한다.

```java
long deleteByGenre(String genre);
```

삭제된 레코드 목록을 반환하는 삭제 쿼리는 `List/Set<entity>`를 반환한다.

```java
List<Author> removeByGenre(String genre);
```

내부적으로 SELECT와 DELETE를 사용한다.
