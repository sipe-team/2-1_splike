# 기본 타임스탬프

아래와 같이 BaseEntity 클래스를 생성하여 타임스탬프를 기록

```java
@MappedSuperclass
public abstract class BaseEntity<U> {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    protected Long id;
    @CreationTimestamp
    protected LocalDateTime created; // 생성 일시
    @UpdateTimestamp
    protected LocalDateTime lastModified; // 수정 일시
    @CreatedBy
    protected U createdBy; // 등록한 사용자
    @ModifiedBy
    protected U lastModifiedBy; // 마지막으로 수정한 사용자
}

@Entity
public class Book extends BaseEntity<String> {
    private String title;
    private String isbn;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;
}
```

`AuditorAware` 를 구현해서 JPA가 현재 로그인한 사용자가 누구인지 알 수 있도록 함

보통 SpringSecurity를 통해 구현할 수 있음 (아래는 더미 코드)

```java
public class AuditorAwareImpl implements AuditorAware<String> {
    @Override
    public Optional<String> getCurrentAuditor() {
        // use Spring Security to retrieve the currently logged-in user(s)
        return Optional.of(Arrays.asList("mark1990", "adrianm", "dan555").get(new Random().nextInt(3)));
    }
}
```

# 하이버네이트 Envers

데이터 로깅, 엔티티 이력 테이블 및 쿼리 등을 제공하는 모듈

### 설정

의존성 추가

```xml
<dependency>
	<groupId>org.hibernate</groupId>
	<artifactId>hibernate-envers</artifactId>
</dependency>
```

엔티티 어노테이션 추가

```java
@Entity
@Audited  // 추가
@AuditTable("author_audit")  // 히스토리 테이블 정의
public class Author {
  // ...
}
```

위와 같이 설정하면 아래와 같이 테이블이 생성된다. (PRD 환경에서는 수동으로 생성)

```sql

CREATE TABLE `author_audit` (
    `id` BIGINT NOT NULL,
    `rev` INTEGER NOT NULL,
    `revtype` TINYINT,
    `revend` INTEGER,
    `age` INTEGER NOT NULL,
    `genre` VARCHAR(255),
    `name` VARCHAR(255),
    PRIMARY KEY (`id`, `rev`));
ALTER TABLE `author_audit` ADD CONSTRAINT FK_AUTHOR_AUDIT_REV FOREIGN KEY (`rev`) REFERENCES revinfo(`rev`);
ALTER TABLE `author_audit` ADD CONSTRAINT FK_AUTHOR_AUDIT_REVEND FOREIGN KEY (`revend`) REFERENCES revinfo(`rev`);
```

- `revtype` 필드
    - 0 또는 ADD: 행이 추가됨
    - 1 또는 MOD: 행이 수정됨
    - 2 또는 DEL: 행이 삭제됨

### 조회

```java
@Transactional(readOnly = true)
public void queryEntityHistory() {
    AuditReader reader = AuditReaderFactory.get(em);

    AuditQuery queryAtRev = reader.createQuery().forEntitiesAtRevision(Book.class, 3);
    System.out.println("Get all Book instances modified at revision #3:");
    System.out.println(queryAtRev.getResultList());

    AuditQuery queryOfRevs = reader.createQuery().forRevisionsOfEntity(Book.class, true, true);
    System.out.println("\nGet all Book instances in all their states that were audited:");
    System.out.println(queryOfRevs.getResultList());
}
```

# 영속성 컨텍스트 조회

```java
@RequiredArgsConstructor
@Service
public class BookstoreService {
    @PersistenceContext
    private final EntityManager entityManager;
    private final AuthorRepository authorRepository;

    @Transactional
    public void sqlOperations() {
        briefOverviewOfPersistentContextContent();

        Author author = authorRepository.findByName("Joana Nimar");
        briefOverviewOfPersistentContextContent();

        author.getBooks().get(0).setIsbn("not available");
        briefOverviewOfPersistentContextContent();

        authorRepository.delete(author);
        authorRepository.flush();
        briefOverviewOfPersistentContextContent();

        Author newAuthor = new Author();
        newAuthor.setName("Alicia Tom");
        newAuthor.setAge(38);
        newAuthor.setGenre("Anthology");

        Book book = new Book();
        book.setIsbn("001-AT");
        book.setTitle("The book of swords");

        newAuthor.addBook(book); // use addBook() helper

        authorRepository.saveAndFlush(newAuthor);
        briefOverviewOfPersistentContextContent();
    }

    // 현 시점의 영속성 컨텍스트 안에 있는 데이터를 출력하는 메소드
    private void briefOverviewOfPersistentContextContent() {
        System.out.println("\n-----------------------------------------------------");

        org.hibernate.engine.spi.PersistenceContext persistenceContext = getPersistenceContext();

        int managedEntities = persistenceContext.getNumberOfManagedEntities();
        int collectionEntriesSize = persistenceContext.getCollectionEntriesSize();

        System.out.println("Total number of managed entities: " + managedEntities);
        System.out.println("Total number of collection entries: " + collectionEntriesSize);

        // getEntitiesByKey() will be removed and probably replaced with #iterateEntities()
        Map<EntityKey, Object> entitiesByKey = persistenceContext.getEntitiesByKey();

        if (!entitiesByKey.isEmpty()) {
            System.out.println("\nEntities by key:");
            entitiesByKey.forEach((key, value) -> System.out.println(key + ": " + value));

            System.out.println("\nStatus and hydrated state:");
            for (Object entry : entitiesByKey.values()) {
                EntityEntry ee = persistenceContext.getEntry(entry);
                System.out.println("Entity name: " + ee.getEntityName()
                                + " | Status: " + ee.getStatus()
                                + " | State: " + Arrays.toString(ee.getLoadedState()));
            }
        }

        if (collectionEntriesSize > 0) {
            System.out.println("\nCollection entries:");
            persistenceContext.forEachCollectionEntry(
                    (k, v) -> System.out.println("Key:" + k + ", Value:" + (v.getRole() == null ? "" : v)), false);
        }
        System.out.println("-----------------------------------------------------\n");
    }

    // 영속성 컨텍스트를 불러오는 메소드
    private org.hibernate.engine.spi.PersistenceContext getPersistenceContext() {
        SharedSessionContractImplementor sharedSession = entityManager.unwrap
		        (SharedSessionContractImplementor.class);
        return sharedSession.getPersistenceContext();
    }
}
```

출력 예시 (위 예제에서 마지막으로 호출된 `briefOverviewOfPersistentContextContent`)

```java
Total number of managed entities: 2
Total number of collection entries: 1
Entities by key:

EntityKey[com.bookstore.entity.Book#5] :
    Book{id=5, title=The book of swords, isbn=001-AT}
EntityKey[com.bookstore.entity.Author#5]:
    Author {id=5, name-Alicia Tom, genre=Anthology, age=38}

Status and hydrated state:
Entity name: com.bookstore.entity.Book
  | Status: MANAGED
  | State: [Author{id=5, name=Alicia Tom, genre=Anthology, age=38},
            001-AT, The book of swords]
Entity name: com. bookstore.entity. Author
  | Status: MANAGED
  | State: [38, [Book{id=5, title-The book of swords, isbn=001-AT)],
            Anthology, Alicia Tom]

Collection entries:
Key: [Book{id=5, title=The book of swords, isbn=001-AT}],
Value: CollectionEntry[com.bookstore.entity.Author.books#5]
    -›[com.bookstore. entity.Author.books#5]
```