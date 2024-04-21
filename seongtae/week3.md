# Week 3 스터디 내용

- 낙관적 잠금 회피하기 주제 중 항목 136, 137(지난 주에 못 끝냄)
- 가장 효율적인 쿼리 사용하지 않기 주제 중 항목 41 ~ 45

# 항목 136: PESSIMISTIC_READ/WRITE 작동 방식

- 공유 잠금(=읽기 잠금): 여러 프로세스가 동시에 읽기를 허용하고 쓰기를 허용하지 않는다.
- 배타적 잠금(=쓰기 잠금): 쓰기 작업이 진행중인 동안 읽기와 쓰기 모두 허용하지 않는다.

아래와 같이 쿼리 혹은 레포지터리 메서드 단에서 공유 잠금 혹은 배타적 잠금을 획득할 수 있다.

```java
@Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Override
  @Lock(LockModeType.PESSIMISTIC_READ)
  // 또는 @Lock(LockModeType.PESSIMISTIC_WRITE)
  Optional<Author> findById(Long id);
}
```

잠금 획득 방식은 벤더사 마다 다른데, Dialect를 활용해 적절한 구문을 선택한다.

이와 관련된 시나리오는 다음과 같다.

1. 트랜잭션 A는 ID가 1인 저자를 가져온다.
2. 트랜잭션 B는 동일한 저자를 가져온다.
3. 트랜잭션 B는 저자의 장르를 수정한다.
4. 트랜잭션 B가 커밋한다.
5. 트랜잭션 A가 커밋한다.

## PESSIMISTIC_READ

PESSIMISTIC_READ 맥락에서 해당 시나리오는 아래 과정을 거친다.

1. 트랜잭션 A는 ID가 1인 저자를 가져오고 공유 잠금을 획득한다.
2. 트랜잭션 B는 동일한 저자를 가져오고 공유 잠금을 획득한다.
3. 트랜잭션 B는 저자의 장르를 수정하려고 한다.
4. 트랜잭션 A가 공유 잠금을 유지하는 한 해당 행을 수정하기 위한 잠금을 획득할 수 없기 때문에 트랜잭션 B는 타임아웃된다.
5. 트랜잭션 B로 인해 QueryTimeoutException이 발생한다.

### MYSQL Dialect(InnoDB)

MySQL5 MyISAM 엔진은 쓰기를 방지하지 않기에 사용하지 말 것.

```sql
-- 트랜잭션 A는 id가 1인 저자를 가져오고 공유 잠금을 획득
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.id = 1 FOR SHARE

-- 트랜잭션 B는 동일한 저자를 가져오고 공유 잠금을 획득
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.id = 1 FOR SHARE

-- 트랜잭션 B는 저자의 장르를 업데이트하려고 함
-- 트랜잭션 A가 공유 잠금을 보유하고 있어 트랜잭션 B는 해당 행 수정을 위한
-- 잠금을 획득할 수 없으며, 이로 인해 트랜잭션 B는 타임아웃됨
UPDATE author
SET age = 23,
    genre = "Horror",
    name = "Mark Janel"
WHERE id = 1

-- 트랜잭션 B는 QueryTimeoutException을 발생시킴
-- org.springframework.dao.QueryTimeoutException
-- Caused by: org.hibernate.QueryTimeoutException
```

### PostgreSQL에서의 PostgreSQL95Dialect

```sql
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.id = ? FOR SHARE
```

## PESSIMISTIC_WRITE

PESSIMISTIC_WRITE 맥락에서 해당 시나리오는 다음과 같은 과정을 따른다.

1. 트랜잭션 A는 ID가 1인 저자를 가져오고 배타적 잠금을 획득한다.
2. 트랜잭션 B는 ID가 1인 저자의 장르를 Horror로 수정하려 한다. 즉, 해당 저자를 가져오고 배타적 잠금을 얻으려 시도한다.
3. 트랜잭션 A가 배타적 잠금을 갖고 있는 한 해당 행을 수정하기 위한 잠금을 획득할 수 없어 트랜잭션 B는 타임아웃된다.
4. 트랜잭션 B로 인해 QueryTimeoutException이 발생한다.

### MYSQL Dialect(InnoDB)

MySQL5 MyISAM 엔진은 실제로 배타적 잠금을 획득하지 않으므로 사용하지 말 것.

```sql
-- 트랜잭션 A는 id가 1인 저자를 가져오고 배타적 잠금을 획득
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.id = 1 FOR UPDATE

-- 트랜잭션 B는 id가 1인 저자의 장르를 Horror로 업데이트하려고 함
-- 이를 위해 저자를 가져오고 배타적 잠금을 획득하려고 시도
-- 트랜잭션 A가 배타적 잠금을 보유하고 있어 트랜잭션 B는 해당 행 수정을 위한
-- 잠금을 획득할 수 없으며, 이로 인해 트랜잭션 B는 타임아웃됨
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.id = ? FOR SHARE

-- 트랜잭션 B는 QueryTimeoutException를 발생시킴
-- org.springframework.dao.QueryTimeoutException
-- Caused by: org.hibernate.QueryTimeoutException
```

### PostgreSQL에서의 PostgreSQL95Dialect

```sql
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.id = ? FOR UPDATE
```

# 항목 137: PESSIMISTIC_WRITE가 UPDATE/INSERT 및 DELETE에서 작동하는 방식

```java
@Repository
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Override
  @Lock(LockModeType.PESSIMISTIC_WRITE)
  Optional<Author> findById(Long id);

  @Lock(LockModeType.PESSIMISTIC_WRITE)
  List<Author> findByAgeBetween(int start, int end);

  @Modifying
  @Query("UPDATE Author SET genre = ?1 WHERE id = ?2")
  void updateGenre(String genre, long id);
}
```

- UPDATE와 DELETE의 경우 한 트랜잭션이 배타적 잠금을 갖고 있을 때 다른 트랜잭션은 대기한다.

## INSERT 트리거

일반적으로 배타적 잠금의 경우에도 INSERT문 처리가 가능하다.

1. 트랜잭션 A는 findByAgeBetween()을 통해 나이가 40~50세인 모든 저자를 선택하고 배타적 잠금을 획득한다.
2. 트랜잭션 A가 실행되는 동안 트랜잭션 B는 2초 후에 시작해 새로운 저자 등록을 시도한다.

### REPEATABLE_READ와 함께 MySQL에서 INSERT 트리거

기본 격리 수준인 REPEATABLE_READ의 경우 잠긴 항목 범위에 대한 INSERT 문을 방지할 수 있는 MySQL은 예외가 된다.

```sql
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.age BETWEEN ? AND ? FOR UPDATE

-- Locking for 10s

INSERT INTO author (age, genre, name)
  VALUES (?, ?, ?)

-- Releasing lock ...
```

트랜잭션 B는 트랜잭션 A가 커밋된 후에만 등록을 트리거한다. 즉, 트랜잭션 A가 배타적 잠금을 해제할 때까지 차단된다.

### READ_COMMITTED와 함께 MySQL에서 INSERT 트리거

```sql
SELECT
  author0_.id AS id1_0_0_,
  author0_.age AS age2_0_0_,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
FROM author author0_
WHERE author0_.age BETWEEN ? AND ? FOR UPDATE

-- Locking for 10s

INSERT INTO author (age, genre, name)
  VALUES (?, ?, ?)

-- Second transaction committed!
-- Releasing lock ...
```

# 항목 41: 모든 왼쪽 엔티티를 가져오는 방법

- JOIN FETCH는 INNER JOIN으로 변환되기 때문에 결과 세트는 쿼리 실행문 오른쪽 참조 엔티티 또는 테이블과 일치하는 왼쪽 엔티티 또는 테이블의 행을 포함한다.
- LEFT JOIN을 통해서는 일반 SQL에서 실행문 왼쪽의 엔티티 또는 테이블의 모든 행을 가져올 수도 있지만 동일한 SELECT에서 연관된 컬렉션을 가져오지 않는다.
- JOIN FETCH와 LEFT JOIN을 합쳐 LEFT JOIN FETCH를 통해 단점을 제거할 수 있다.

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Query(value = "SELECT a FROM Author a LEFT JOIN FETCH a.books")
  List<Author> fetchAuthorWithBooks();
}
```

```sql
SELECT
  author0_.id AS id1_0_0_,
  books1_.id AS id_1_1_,
  author0_.age AS age2_0_0,
  author0_.genre AS genre3_0_0_,
  author0_.name AS name4_0_0_,
  books1_.author_id AS author_i4_1_1_,
  books1_.isbn AS isbn2_1_1_,
  books1_.title AS title3_1_1_,
  books1_.author_id AS author_i4_1_0__,
  books1_.id AS id1_1_0__
FROM author author0_
LEFT OUTER JOIN book books1_
  ON author0_.id = books1_.author_id
```

# 항목 42: 관련 없는 엔티티로부터 DTO를 가져오는 방법

- 명시적인 연관관계가 없는 엔티티지만 같은 컬럼을 갖고 있는 엔티티의 경우

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Query(value = "SELECT a.name AS name, b.title AS title "
         + "FROM Author a INNER JOIN Book b ON a.name = b.name "
         + "WHERE b.price = ?1")
  List<BookstoreDto> fetchAuthorNameBookTitleWithPrice(int price);
}
```

# 항목 43: JOIN문 작성 방법

- INNER: 데이터가 두 테이블 모두에 있는 경우에 유용
- OUTER
  - LEFT OUTER JOIN: 왼쪽 테이블에 있는 데이터 가져오기
  - RIGHT OUTER JOIN: 오른쪽 테이블에 있는 데이터 가져오기
  - FULL OUTER JOIN: 두 테이블 중 하나라도 있는 데이터 가져오기
  - CROSS JOIN: 모든 데이터에 대한 모든 데이터를 가져온다. ON 또는 WHERE 절이 없는 CROSS JOIN은 카테시안 곱을 제공
- SQL JOIN 문은 유명한 LazyInitializationException을 완화하는 가장 좋은 방법

## INNER JOIN

테이블 author가 A, 테이블 book이 B라고 가정하면 INNER JOIN은 다음과 같다.

```java
@Query(value = "SELECT b.title AS title, a.name AS name "
       + "FROM Author a INNER JOIN a.books b")
List<AuthorNameBookTitle> findAuthorsAndBooksJpql();
```

## LEFT JOIN

```java
@Query(value = "SELECT b.title AS title, a.name AS name "
       + "FROM Author a LEFT JOIN a.books b")
List<AuthorNameBookTitle> findAuthorsAndBooksJpql();
```

## RIGHT JOIN

```java
@Query(value = "SELECT b.title AS title, a.name AS name "
       + "FROM Author a RIGHT JOIN a.books b")
List<AuthorNameBookTitle> findAuthorsAndBooksJpql();
```

## CROSS JOIN

CROSS JOIN에 ON 또는 WHERE 절이 없으면 카테시안 곱 결과를 반환

```java
@Query(value = "SELECT b.title AS title, f.formatType AS formatType "
       + "FROM Book b, Format f")
List<BookTitleAndFormatType> findBooksAndFormatsJpql();
```

## -to-one 연관관계 implicit JOIN 문 주의

INNER JOIN이 아니라 CROSS JOIN을 실행한다.

```java
@Query(value = "SELECT b.title AS title, b.author.name "
       + "AS name FROM Book b")
List<AuthorNameBookTitle> findBooksAndAuthorsJpql();
```

이 암묵적인 JOIN은 INNER JOIN이 아닌 WHERE 절의 CROSS JOIN을 만든다.

```
명시적으로 JOIN문을 사용하는 것이 좋다. 엔티티를 가져오는 경우 JOIN FETCH를 사용한다.
```

## FULL JOIN

```java
@Query(value = "SELECT b.title AS title, a.name AS name "
       + "FROM Author a FULL JOIN a.books b")
LIST<AuthorNameBookTitle> findAuthorsAndBooksJpql();
```

# 항목 44: JOIN 페이지네이션 방법

JOIN + 프로젝션(DTO) 사용

```java
public interface AuthorBookDto {
    String getName();
    int getAge();
    String getTitle();
    String getIsbn();
}
```

LEFT JOIN을 사용해 JPQL 작성

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long> {

  @Query(value = "SELECT a.name AS name, a.age AS age,
         b.title AS title, b.isbn AS isbn "
         + "FROM Author a LEFT JOIN a.books b WHERE a.genre = ?1")
  Page<AuthorBookDto> fetchPageOfDto(String genre, Pageable pageable);
}
```

fetchPageOfDto()를 호출하는 서비스 메서드

```java
public Page<AuthorBookDto> fetchPageOfAuthorsWithBooksDtoByGenre(
                                                int page, int size) {
  Pageable pageable = PageRequest.of(page, size,
          Sort.by(Sort.Direction.ASC, "name"));

  Page<AuthorBookDto> pageOfAuthors
         = authorRepository.fetchPageOfDto("Anthology", pageable);

  return pageOfAuthors;
}
```

SELECT COUNT는 SELECT 서브쿼리 또는 COUNT(\*) OVER() 윈도우 함수를 사용해 단일 쿼리로 전환된다. COUNT(\*) OVER()를 활용하려면 AuthorBookDto에 추가 필드를 생성한다.

```java
public interface AuthorBookDto {
    String getName();
    int getAge();
    String getTitle();
    String getIsbn();

    @JsonIgnore
    long getTotal();
}
```

```java
@Transactional(readOnly = true)
@Query(value = "SELECT a.name AS name, a.age AS age, b.title AS title,
       b.isbn AS isbn,"
       + " COUNT (*) OVER() AS total FROM author a LEFT JOIN book b "
       + "ON a.id = b.author_id WHERE a.genre = ?1",
       nativeQuery = true)
List<AuthorBookDto> fetchListOfDtoNative(String genre, Pageable pageable);
```

```java
public Page<AuthorBookDto> fetchPageOfAuthorsWithBooksDtoByGenreNative(
                                                int page, int size) {
  Pageable pageable = PageRequest.of(page, size,
          Sort.by(Sort.Direction.ASC, "name"));

  List<AuthorBookDto> listOfAuthors
         = authorRepository.fetchPageOfDtoNative("Anthology", pageable);

  Page<AuthorBookDto> pageOfAuthors
         = new PageImpl(listOfAuthors, pageable,
         listOfAuthors.isEmpty() ? 0 : listOfAuthors.get(0).getTotal());

  return pageOfAuthors;
}
```

페이지네이션으로 인해서 결과 세트가 잘리는 경향이 있다. 따라서 저자는 도서의 하위 집합만을 가져올 수 있다.

결과 세트 잘림을 방지하려면 어떻게 해야 하는가?

## DENSE_RANK() 윈도우 함수 해결 방안

- DENSE_RANK는 각 그룹 b 내에서 a의 서로 다른 값에 일련번호를 할당하는 함수

```java
@Transactional(readOnly = true)
@Query(value = "SELECT * FROM ("
        + "SELECT *, DENSE_RANK() OVER (ORDER BY name, age) na_rank FROM ("
        + "SELECT a.name AS name, a.age AS age, b.title AS title, b.isbn AS isbn "
        + "FROM author a LEFT JOIN book b ON a.id = b.author_id "
        + "WHERE a.genre = ?1 "
        + "ORDER BY a.name) ab ) ab_r "
        + "WHERE ab_r.na_rank > ?2 AND ab_r.na_rank <= ?3",
        nativeQuery = true)
List<AuthorBookDto> fetchListOfDtoNativeDenseRank(String genre, int start, int end);
```

도서를 자르지 않고 도서를 갖는 처음 두 저자를 가져올 수 있다.
