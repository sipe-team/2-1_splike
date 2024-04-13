# 6장 커넥션과 트랜잭션
## 항목 60: 실제 필요 시점까지 커넥션 획득 지연 방법
1) 커넥션 풀 setAutoCommit false로 설정
2) hibernate.connection.provider_disables_autocommit true로 설정

```java
// spring boot application.propertoes
spring.datasource.hikari-auto-commit=false
spring.jpa.properties.hibernate.connection.provider_disables_autocommit=true
```
---
### JDBC (Java Database Connectivity)
데이터베이스를 연결하기 위한 Java 표준 SQL 인터페이스.
DBMS 종류에 상관없이 같은 방식으로 연결 할 수 있음.

### Connection Pool
pool에 데이터베이스와 연결 객체(Connection)을 미리 여러개 만들어 둠.
요청 시 미리 만들어놓은 Connection을 사용하고, 반납하는 형식으로 DB 자원 관리.

#### HikariCP
spring boot 2.0 default



---

## 항목64: 리포지터리 인터페이스에서 @Transactional을 사용하는 이유와 방법

처리기간이 짧은 트랙잭션을 처리??

리포지터리 성능 업
1) 도메인 클래스별 리포지터리 인터페이스를 정의한다.






