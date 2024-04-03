# 내용 정리

## 항목 1: `@OneToMany` 연관관계를 효과적으로 구성하는 방법

<img width="730" alt="image" src="https://github.com/sipe-team/2-1_splike/assets/83271772/7bafa13f-2862-44ba-89e3-e800ac912976">

- 양방향 지연 연관관계 예시 ERD
- author는 여러 개의 book을 발행할 수 있으므로 author가 부모 측(@OneToMany), book이 자식 측(@ManyToOne)이 된다.
- @ManyToOne 연관관계는 외래키 열을 영속성 컨텍스트(데이터 베이스와 애플리케이션 중간에서 캐시 역할을 하는 환경)

```md
일반적인 규칙으로 단방향 연관관계보다 양방향 연관관계 사용
```

### 항상 부모 측에서 자식 측으로 전이(Cascade)를 사용

- <u>**자식 측에서 부모 측으로의 전이(cascading)은 잘못된 관행**</u>
  - 자식은 부모 없이 존재할 수 없으며, 부모가 자식을 전이시키는 것이다.
  - ex) 저자가 존재하지 않은 책은 웬만해서 없다. 저자가 탄생하며 해당 저자의 책이 같이 생성될 수는 있다.
- Author에서 Book으로 전이될 수 있도록 Author 엔티티에 전이 타입을 추가한다.

```java
@OneToMany(cascade = CascadeType.ALL)
```

### 부모 측에 mappedBy 지정

- 부모 측에 설정되는 mappedBy 속성은 양방향 연관관계의 특성을 부여
- 자식 측이 어떤 엔티티로 매핑되어야 하는지를 지정
  - ex) 해당 도서가 어떤 저자와 이어져야 하는지 저자가 책에 명시

```java
@OneToMany(cascade = CascadeType.ALL,
                mappedBy = "author")
```

### 부모 측에 orphanRemoval 지정

- `orphanRemoval`이란 더 이상 참조되지 않는 자식들을 삭제하는 설정
- 자식 측이 참조없이 존재할 수 없는 의존 객체를 삭제하기에 용이하다.
  - ex) 유저와 유저 정보 간에 유저 정보는 부모 측이 존재하지 않으면 존재할 수 없다.

```java
@OneToMany(cascade = CascadeType.ALL,
                mappedBy = "author",
                orphanRemoval = true)
```

### 연관관계의 양측을 동기화로 유지

- 부모 측과 자식 측은 동기화 상태로 유지되어야야 한다.
- 부모 측에 Helper 메서드를 사용해서 addChild(child 프로퍼티의 세터 느낌)이나 removeChild 등

```java
  public void addBook(Book book) {
    this.books.add(book);
    book.setAuthor(this);
  }

  public void removeBook(Book book) {
    book.setAuthor(null); // 자식 측의 부모 엔티티 연관 제거
    this.books.remove(book); // 부모 측 자식 배열에서 해당 자식 엔티티 연관 제거
  }

  public void removeBooks() {
    Iterator<Book> iterator = this.books.iterator();

    while (iterator.hasNext()) {
      Book book = iterator.next();

      book.setAuthor(null);
      iterator.remove();
    }
  }
```

### `equals()`와 `hashCode()` 오버라이딩

- 보통 데이터베이스의 식별자를 사용해서 두 엔티티가 같은지 비교한다. (각 식별자는 unique한 값이기 때문)
- 데이터베이스의 식별자는 auto-increment로 자동 생성하며, 이 자동 생성된 데이터베이스 식별자를 기반으로 `equals()`와 `hashCode()`를 오버라이딩한다.
- 식별자가 초기화되기 전에는 null로 존재하므로 null 검사를 수행하고, `hashCode()`는 상수 값을 반환해야 한다.

💡 코틀린의 data class는 내부적으로 `equals()`와 `hashCode()`를 만들어 주어 해당 메서드는 동등 연산을 할 때 데이터베이스 식별자를 사용하지 않는다. 따라서, data class가 아니라 일반 class로 사용하는 것을 권장한다.

```java
  @Override
  public boolean equals(Object obj) {
    // ...
    return id != null && id.equals(((Book) obj).id);
  }

  @Override
  public int hashCode() {
    return 2021; // 랜덤으로 설정한 값
  }
```

### 연관관계 양측에서 지연 로딩 사용

> 김영한님 강의에서 매우매우 강조한 내용 중 하나

- 기본적으로 부모 측 엔티티를 가져오더라도 자식 측 엔티티는 가져오지 않음 -> `Lazy Loading`!
- ⛔️ 하지만 자식 측 엔티티는 부모 측 엔티티를 즉시 가져옴 -> `Eager Loading`!
- 자식 측 엔티티에서는 명시적으로 지연 로딩을 설정하고 쿼리 기반에서만 즉시 가져오도록 하는 것이 좋다.

💡 @xToOne 관계는 기본이 즉시 로딩이므로 LAZY 설정하는 것이 좋다.  
💡 저희 회사 자바 관련 시니어 개발자분께서는 데이터가 별로 없으면 편하게 Eager loading 쓰라고 하셨어요

```java
@ManyToOne(fetch = FetchType.LAZY)
```

#### 즉시 로딩이 문제인 이유

- JPQL에서 N+1 문제를 일으킨다.
- 책을 10개 조회했으면 해당 책의 저자 10명을 추가로 조회하게 되므로, 책 조회 쿼리 1번에 각각의 저자 조회 쿼리 10번이 발생해 N + 1 (정확히는 1 + N 이긴 함!)

### `toString()` 오버라이딩 방법에 주의

- `toString()`을 오버라이딩할 때는 엔티티가 DB로부터 가져오는 기본 속성만 포함해야 한다.
- 지연 속성 및 연관관계는 초기화되지 않았으므로 주의

```java
@Override
public String toString() {
  return "Author{" + "id=" + id + ", name=" + name
      + ", genre=" + genre + ", age=" + age + "}";
}
```

### `@JoinColumn`을 사용해 조인 컬럼 지정

- 부모 측(Author) 엔티티에 대한 외래키를 갖는데, 이 칼럼은 의도한 이름으로 지정.

```java
@JoinColumn(name = "author_id")
```

## 항목 2: 단방향 `@OneToMany` 연관관계를 피해야 하는 경우

🤔 보통 OneToMany는 지양하라고 하던데 양방향은 괜찮고 단방향은 지양해야 하나 보네요?

Author와 Book 엔티티가 아래와 같이 매핑된 단방향 `@OneToMany` 연관관계로 가정

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
private List<Book> books = new ArrayList<>();
```

누락된 @ManyToOne 연관관계는 아래와 같이 별도의 연결 테이블(author_books)을 생성한다.

<img width="999" alt="image" src="https://github.com/sipe-team/2-1_splike/assets/83271772/fc0d5754-8b25-4c34-8b52-3eac3fcb30ff">

- 2개의 외래키가 있어 인덱싱에서 @OneToMany의 경우보다 더 많은 메모리를 사용
- 3개의 테이블이 있으므로 쿼리 작업에도 영향

### 일반적인 단방향 @OneToMany

#### 저자와 해당 도서 등록

```java
    @Transactional
    public void insertAuthorWithBooks() {

        Author jn = new Author();
        jn.setName("Joana Nimar");
        jn.setAge(34);
        jn.setGenre("History");

        Book jn01 = new Book();
        jn01.setIsbn("001-JN");
        jn01.setTitle("A History of Ancient Prague");

        Book jn02 = new Book();
        jn02.setIsbn("002-JN");
        jn02.setTitle("A People's History");

        Book jn03 = new Book();
        jn03.setIsbn("003-JN");
        jn03.setTitle("World History");

        jn.addBook(jn01);
        jn.addBook(jn02);
        jn.addBook(jn03);

        authorRepository.save(jn);
    }
```

해당 메서드를 수행했을 때 SQL INSERT 문을 보면 양방향 연관관계 @OneToMany는 1번의 쿼리였던 것에 비해 단방향 @OneToMany는 3번의 INSERT 쿼리가 추가됐다.

#### 기존 저자의 새로운 도서 등록

```java
    @Transactional
    public void insertNewBook() {

        Author author = authorRepository.fetchByName("Joana Nimar");

        Book book = new Book();
        book.setIsbn("004-JN");
        book.setTitle("History Details");

        author.addBook(book); // use addBook() helper

        authorRepository.save(author);
    }
```

이 메서드를 호출했을 떄 SQL INSERT 문을 보면 기존에 저자가 3권의 책을 생성했다고 치면, DELETE 쿼리 1번과 INSERT 쿼리 4번이 수행된다.

- 새 도서를 추가하고자 JPA 영속성 공급자는 연결 테이블에서 모든 연관 도서를 삭제한 후 메모리에 새 도서를 추가하고 결과를 다시 등록

#### 마지막 도서 삭제

```java
    @Transactional
    public void deleteLastBook() {

        Author author = authorRepository.fetchByName("Joana Nimar");
        List<Book> books = author.getBooks();

        author.removeBook(books.get(books.size() - 1));
    }
```

- 마지막 도서를 삭제하고자 JPA는 연결 테이블에서 모든 관련 도서를 삭제
- 메모리에서 마지막 도서를 제거한 후 나머지 도서를 다시 등록

#### 첫 번째 도서 삭제

```java
    @Transactional
    public void deleteFirstBook() {

        Author author = authorRepository.fetchByName("Joana Nimar");
        List<Book> books = author.getBooks();

        author.removeBook(books.get(0));
    }
```

마지막 도서를 삭제하는 것과 똑같이 동작

> 추가 SQL문이 동적으로 늘어가는 것 + 연결 테이블 외래키 컬럼과 관련된 인덱스 write 연산으로 인한 성능 저하.  
> <u>**💡 단방향 @OneToMany 연관관계는 양방향 @OneToMany 연관관계보다 덜 효율적이다!**</u>

### `@OrderColumn` 사용

- 단방향 @OneToMany 연관관계가 정렬됨
- 컬렉션이 `ORDER BY` 절을 사용해 정렬되도록 연결 테이블의 별도 컬럼으로 항목 인덱스를 구체화하도록 지시한다.

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@OrderColumn(name = "books_order")
private List<Book> books = new ArrayList<>();
```

#### 저자와 도서 등록

`insertAuthorWithBooks()` 서비스 메서드를 통해 스냅숏에서 저자와 연관된 도서를 등록하면 여전히 cascade가 아닌 개별 INSERT 쿼리 수행

#### 기존 저자의 새로운 도서 등록

`insertNewBook()` 서비스 메서드를 통해 새 도서를 등록하면 장점과 단점이 존재

- 장점: 저자의 연관 도서를 메모리에서 다시 추가하고자 삭제하지 않았다(N개의 책을 지우고 N+1개를 INSERT 하지 않음!)
- 단점: 여전히 연결 테이블에 추가 INSERT 문이 있다는 것

#### 마지막 도서 삭제

`deleteLastBook()` 을 통해 마지막 도서를 삭제

- JPA 영속성 공급자(하이버네이트)가 삭제되지 않은 나머지 책을 메모리에서 다시 추가하고자 연관된 모든 도서를 삭제하지 않음
  - 불필요한 삭제가 발생하지 않았다.
- 여전히 연결 테이블에 호출되는 추가 DELETE 존재

#### 첫 번째 도서 삭제

- 컬렉션의 끝에서 멀어질수록 업데이트가 많이 일어나므로 `@OrderColumn`의 이점이 작이진다.
- 첫 번째 도서를 삭제하면 연결 테이블에서 DELETE가 발생하고 컬렉션의 메모리 순서를 유지하기 위해 UPDATE 문이 뒤따른다.

> 여전히 양방향 `@OneToMany`보다 낫지 않다.

### `@JoinColumn` 사용

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "author_id")
private List<Book> books = new ArrayList<>();
```

@JoinColumn을 지정하면 @OneToMany 연관관계가 자식 테이블 외래키를 제어할 수 있음을 하이버네이트에 지시한다.

<img width="730" alt="image" src="https://github.com/sipe-team/2-1_splike/assets/83271772/7bafa13f-2862-44ba-89e3-e800ac912976">

💡 연결 테이블이 줄어들었다!

#### 저자와 도서 등록

등록되는 도서 각각이 author_id 값을 설정해야 하므로 UPDATE를 계속 호출한다.

- 불필요한 UPDATE Query 호출

#### 기존 저자의 새로운 도서 등록

위와 마찬가지로 UPDATE로 book의 author_id를 수정해야 한다.

- 불필요한 UPDATE Query 호출

#### 마지막 도서 삭제

JPA 영속성 공급자(하이버네이트)는 author_id를 null로 설정해 저자로부터 도서 연관관계를 끊음

=> orphanRemoval 설정으로 고아가 된 자식 엔티티는 삭제된다.

#### 첫 번째 도서 삭제

UPDATE를 여전히 사용한다.

> `@JoinColumn`을 추가하면 일반 단방향 `@OneToMany` 보다는 좋지만 여전히 양방향 `@OneToMany` 연관관계보다는 낫지 않다.  
> <u>**추가 UPDATE 문이 필요 없는데 발생한다.**</u>

### 요약

> 1. 부모측에 cascade, mappedBy, orphanRemoval을 지정하자.
> 2. 연관관계의 양측에 지연 로딩을 사용하자.
> 3. 단방향 `@OneToMany`보다 양방향 `@OneToMany`를 지향하자
