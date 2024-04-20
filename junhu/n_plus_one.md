## 항목6: CascadeType.REMOVE 및 orphanRemoval=true를 사용해 하위 엔티티 제거를 피해야 하는 이유와 시기

다음과 같이 작성된 양방향 지연 @OneToMany 연관관계를 갖는 Author와 Book 엔티티를 사용하자.

```java
// Author의 부분
@OneToMany(cascade = CascadeType.ALL, mappedBy = "author", orphanRemoval = true)
private List<Book> books = new ArrayList<>();

// Book의 부분
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "author_id")
private Author author;
```

Author 엔티티 삭제는 연관된 Book 엔티티로 자동 전이된다. 이는 CascadeType.REMOVE 또는 orphanRemoval=true가 하나라도 설정돼 있으면 처리가 되는데, 두 설정이 중복된다는 것이다.

그렇다면 어떻게 다른가? 다음과 같이 저자로부터 도서를 연결 해제하는 데 사용하는 도우미 메서드가 있다고 생각해보자.

```java
public void removeBook(Book book) {
	book.setAuthor(null);
	this.books.remove(book);
}
```

또는 모든 도서를 저자로부터 연결 해제하는 경우다.

```java
public void removeBooks() {
    Iterator<Book> iterator = this.books.iterator();

    while (iterator.hasNext()) {
        Book book = iterator.next();

        book.setAuthor(null);
        iterator.remove();
    }
}

```

orphanRemoval=true가 있는 상태에서 removeBook() 메서드를 호출하면 DELETE 문을 사용해 도서가 자동으로 삭제된다. orphanRemoval=false가 지정돼 있으면 UPDATE문이 호출된다. 도서의 연결 해제 자체는 삭제 처리가 아니기 때문에 CascadeType.REMOVE 존재 여부는 상관없다. 따라서 orphanRemoval=true는 오너 엔티티 참조 없이는 존재하지 않아야 할 엔티티를 정리하는데 유용하다. 

1. CascadeType.REMOVE - 부모 엔티티가 삭제되면 자식 엔티티도 같이 삭제

```java
// not efficient
@Transactional
public void deleteViaCascadeRemove() {
    Author author = authorRepository.findByName("Joana Nimar");

    authorRepository.delete(author);
}
```

1. orphanRemoval=true - 자식 엔티티의 authorId가 null이 되어 고아가 되어 삭제

```java
    // not efficient
    @Transactional
    public void deleteViaOrphanRemoval() {
        Author author = authorRepository.findByNameWithBooks("Joana Nimar");

        author.removeBooks();
        authorRepository.delete(author);
    }
```

그러나 둘다 저자 1명에 책이 5개 있을 경우, 책 5개 삭제 쿼리가 각각 실행되고 저자 1명 삭제 쿼리가 실행된다. 그래서 둘다 성능적으로 좋지않다. 하지만 위의 방식을 사용해서는 부모와 자식을 위한 낙관적 잠금의 장점을 누릴 수 있다.

낙관적 락과 비관적 락: [https://velog.io/@bagt/Database-낙관적-락-비관적-락](https://velog.io/@bagt/Database-%EB%82%99%EA%B4%80%EC%A0%81-%EB%9D%BD-%EB%B9%84%EA%B4%80%EC%A0%81-%EB%9D%BD)

### 성능 챙기기

1. 영속성 컨텍스트에 이미 로드된 저자 삭제
    1. 하나의 저자만 영속성 컨텍스트에 로드된 경우, authorId로 book을 먼저 삭제후 저자를 삭제한다.
        
        ```java
        **@Transactional
        @Modifying(flushAutomatically = true, clearAutomatically = true)
        @Query("DELETE FROM Book b WHERE b.author.id = ?1")
        int deleteByAuthorIdentifier(Long id);**
        ```
        
        ```java
            @Transactional
            @Modifying(flushAutomatically = true, clearAutomatically = true)
            @Query("DELETE FROM Author a WHERE a.id = ?1")
            int deleteByIdentifier(Long id);
        ```
        
    2. 여러 저자가 영속성 컨텍스트에 로드된 경우
        
        ```java
        @Transactional
        @Modifying(flushAutomatically = true, clearAutomatically = true)
        @Query("DELETE FROM Book b WHERE b.author IN ?1")
        int deleteBulkByAuthors(List<Author> authors);
        ```
        
        ```java
        // More Author are loaded in the Persistent Context
        @Transactional
        public void deleteViaBulkIn() {
            List<Author> authors = authorRepository.findByAge(34);
        
            bookRepository.deleteBulkByAuthors(authors);
            authorRepository.deleteInBatch(authors);
        }
        ```
        
    3. 한 저자와 관련 도서들이 영속성 컨텍스트에 로드된 경우 - 위의 케이스는 book이 로드가 되어 있지 않았을때 사용가능한 케이스
        
        ```java
        // One Author and his associated Book have been loaded in the Persistence Context
        @Transactional
        public void deleteViaDeleteInBatchX() {
            Author author = authorRepository.findByNameWithBooks("Joana Nimar");
        
            bookRepository.deleteInBatch(author.getBooks());
            authorRepository.deleteInBatch(List.of(author));
        
            // later on, we forgot that this author was deleted
            // throw ObjectOptimisticLockingFailureException
            author.setGenre("Anthology");
        }
        ```
        
        이 방식의 단점은 영속성 컨텍스트를 플러시나 클리어하지 않는 메서드를 사용한다. 그래서 삭제 후 클리어가 필요하다. 그렇지 않으면 setMethod에서 예외가 발생한다.
        
        실질적으로 수정은 영속성 컨텍스트에 포함된 author 엔티티를 변경하지만 이 컨텍스느는 저자가 데이터 베이스에서 삭제됐기 때문에 유효하지 않다. 즉, 저자와 연관된 도서를 데이터베이스에서 삭제한 후에도 현재 영속성 컨텍스트에 계속 존재하게 된다. deleteInBatch를 통해 수행된 삭제를 영속성 컨텍스트는 인식하지 못하므로 영속성 컨텍스트가 클리어되게 하려면 deleteInBatch를 재정의해 @Modifying(clearAutomatically=true)를 추가해야 한다.
        
        [https://velog.io/@gruzzimo/JPA-Modifying의-flushAutomatically-옵션은-언제-쓰지](https://velog.io/@gruzzimo/JPA-Modifying%EC%9D%98-flushAutomatically-%EC%98%B5%EC%85%98%EC%9D%80-%EC%96%B8%EC%A0%9C-%EC%93%B0%EC%A7%80)
        

## 항목7. JPA 엔티티 그래프를 통해 연관관계를 가져오는 방법

엔티티 그래프는 jpa2.1에서 도입됐으며 n+1 문제를 해결해 엔티티 로드 성능 개선에 도움이 된다. 개발자는 엔티티와 관련된 연관관계와 하나의 select 문에 로드돼야 할 기본적인 필드들을 지정하고 해당 엔티티에 대한 여러 엔티티 그래프를 정의해 다른 엔티티를 연결할 수 있으며 하위 그래프를 사용해 복잡한 페치 플랜을 만들 수 있다. 

@NamedEntityGraph로 엔티티 그래프 정의

@NamedEntityGrpah 어노테이션은 에티티에 정의되는데, 엔티티 그래프의 고유한 이름과 엔티티 그래프를 가져올 때 포함할 속성을 지정한다. 속성은 기본 필드와 연관관계가 지정된다.

```java
@Entity
@NamedEntityGraph(
    name = "author-books-graph",
    attributeNodes = {
        @NamedAttributeNode("books")
    }
)
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
}
```

이제 이 엔티티그래프를 사용하려면 AuthorRepository에 엔티티 그래프를 지정해주어야 한다. 이때 @EntityGraph 어노테이션을 사용한다.

```java
  @EntityGraph(value = "author-books-graph",
      type = EntityGraph.EntityGraphType.FETCH)
  @Override
  List<Author> findAll();
```

엔티티 그래프를 사용하려면 2가지 속성을 알아야 한다. 이는 fetchType에 관한 재정의와 관련이 있다.

1. 페치 그래프: javax.persistence.fetchgraph 속성으로, attributes에 지정된 속성들은 eager, 나머지는 lazy로 고정되어 처리 된다.
2. 로드 그래프: java.persistence.loadgraph 속성으로, attributes에 지정된 속성은 eager, 나머지 속성은 지정된 FetchType이나 기본 FetchType에 따라 처리된다.

**애드혹 엔티티 그래프 - @NamedEntityGraph 사용이 필요없이 바로 사용 가능하다.**

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long>, JpaSpecificationExecutor<Author> {
    @Override
    @EntityGraph(attributePaths = {"books"},
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findAll();
}
```

**EntityManager를 통한 엔티티 그래프 정의**

```java
EntityGrpah entityGraph = entityManager.getEntityGraph("author-books-graph");

Author author = entityManager.creatQuery("SELECT a FRO~~~~)
														 .setHint("javax"persistence.fetchgraph", entityGraph)
														 .getSingleResult();
```

## 항목8. JPA 엔티티 서브 그래프를 통해 연관관계를 가져오는 방법

서브그래프를 사용하면 복잡한 엔티티 그래프를 작성할 수 있다. 예를들면 author-onetomany-book-onetomany-publisher가 있다하자. 엔티티 그래프의 목표는 연관된 도서와 함께 모든 저자 그리고 해당 도서와 연관된 출판사를 가져오는 것이다. 이를 위해 엔티티 서브그래프를 사용한다.

**@NamedEntityGraph 및 @NamedSubgraph 사용**

Author 엔티티에서 @NamedEntityGrpah를 사용해 저자와 연관된 도서를 즉시 로딩하는 엔티티 그래프를 정의하고, @NamedSubgraph로 로드된 도서와 관련된 출판사를 가져오기 위한 엔티티 서브그래프를 정의한다.

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
**}**
```

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long>, JpaSpecificationExecutor<Author> {
    @Override
    @EntityGraph(value = "author-books-publisher-graph",
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findAll();
}
```

**에드혹 엔티티 그래프에서 점 노테이션(.) 사용**

애드혹 엔티티 그래프에서도 서브그래프를 사용할 수 있다.

```java
@Repository
@Transactional(readOnly = true)
public interface AuthorRepository extends JpaRepository<Author, Long>, JpaSpecificationExecutor<Author> {
    @Override
    @EntityGraph(attributePaths = {"books.publisher"},
        type = EntityGraph.EntityGraphType.FETCH)
    List<Author> findAll();
}
```

**EntityManager를 통한 엔티티 서브 그래프 정의**

## 항목 9: 엔티티 그래프 및 기본 속성 처리 방법

만약 저자에서는 id와 name만 가져오고 싶을 때는 다음과 같이 설정하면 된다.

```java
@Entity
@NamedEntityGraph(
    name = "author-books-graph",
    attributeNodes = {
        @NamedAttributeNode("name"),
        @NamedAttributeNode("books")
    }
)
public class Author implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    @Basic(fetch = FetchType.LAZY) // 기본이 eager이기 때문에 패치 그래프에서 가져오지 말아야 할 기본 속성을 추가로 지정한다.
    private String genre;
    @Basic(fetch = FetchType.LAZY)
    private int age;

    @OneToMany(cascade = CascadeType.ALL,
        mappedBy = "author", orphanRemoval = true)
    private List<Book> books = new ArrayList<>();
}
```

사실 이렇게 해도 genre와 age를 불러오는데, 이는 하이버네이트에 @Basic 속성을 기본적으로 무시하기 때문이다. 이 문제를 해결하기 위해선 메이븐에 다음과 같은 플러그인을 추가해야 한다.

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.hibernate.orm.tooling</groupId>
                <artifactId>hibernate-enhance-maven-plugin</artifactId>
                <version>5.6.14.Final</version>
                <executions>
                    <execution>
                        <configuration>
                            <failOnError>true</failOnError>
                            <enableLazyInitialization>true</enableLazyInitialization>
                        </configuration>
                        <goals>
                            <goal>enhance</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```
