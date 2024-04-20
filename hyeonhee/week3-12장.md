PRD 환경은 ddl-auto를 사용하지 말아야 한다.

`ddl-auto: validate`로 설정하고 Flyway 또는 Liquibase를 활용해야 한다.

Flyway를 통해 테이블의 변경 이력을 기록하고, 반영할 수 있다.

### Flyway 설정

의존성 추가

```java
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-mysql</artifactId>
</dependency>
```

### Flyway가 데이터베이스를 생성하도록 설정

`application.properties` 설정

```
# URL에 DB 이름을 설정하지 않음
spring.datasource.url=jdbc:mysql://localhost:3306/

# 생성할 DB 이름 지정
spring.flyway.schemas=bookstoredb
```

Entity 클래스에 데이터베이스 명을 알림

```java
@Entity
@Table(schema = "bookstoredb") // or @Table(catalog = "bookstoredb")
public class Book implements Serializable { ... }

@Entity
@Table(schema = "bookstoredb") // or @Table(catalog = "bookstoredb")
public class Author implements Serializable { ... }
```

### 다양한 Datasource

+ 위의 `@Table` 어노테이션을 통해 여러 데이터베이스 url로 지정할 수 있음

```
app.datasource.ds1.url=jdbc:mysql://localhost:3306/booksdb?createDatabaseIfNotExist=true
app.datasource.ds1.username=bookstore
app.datasource.ds1.password=bookstore
app.datasource.ds1.connection-timeout=50000
app.datasource.ds1.idle-timeout=300000
app.datasource.ds1.max-lifetime=900000
app.datasource.ds1.maximum-pool-size=5
app.datasource.ds1.minimum-idle=5
app.datasource.ds1.pool-name=BooksPoolConnection

app.flyway.ds1.location=classpath:db/migration/booksdb

app.datasource.ds2.url=jdbc:mysql://localhost:3306/authorsdb?createDatabaseIfNotExist=true
app.datasource.ds2.username=bookstore
app.datasource.ds2.password=bookstore
app.datasource.ds2.connection-timeout=50000
app.datasource.ds2.idle-timeout=300000
app.datasource.ds2.max-lifetime=900000
app.datasource.ds2.maximum-pool-size=6
app.datasource.ds2.minimum-idle=6
app.datasource.ds2.pool-name=AuthorsConnectionPool

app.flyway.ds2.location=classpath:db/migration/authorsdb

spring.flyway.enabled=false
spring.jpa.show-sql=true

logging.level.com.zaxxer.hikari.HikariConfig=DEBUG
logging.level.com.zaxxer.hikari=TRACE
```