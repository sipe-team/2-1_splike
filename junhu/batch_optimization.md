# 낮은 성능 배치 처리 구현
항목 46-55

## 항목 46: 스프링 부트 스타일 배치 등록 방법

배치 처리는 INSERT, UPDATE, DELETE 문을 그룹화 할 수 있는 매커니즘으로서 결과적으로 데이터베이스/네트워크 호출 횟수를 크게 줄이는데, 일반적으로 호출 횟수가 적을 수록 성능이 향상된다.

### 배치 처리 활성화 및 JDBC URL 설정

**배치 크기 설정**
배치 크기는 spring.jpa.properties.hibernate.jdbc.batch_size 속성으로 설정한다. 권장 값은 5~30 사이.

**mysql 배치 최적화**
mysql의 배치 처리 성능을 최적화 하는데 사용하는 몇 가지 속성이 있다.

1. rewriteBatchedStatements(JDBC URL 최적화 플래그 속성): 이 속성이 활성화되면 SQL문이 하나의 문자열 버퍼로 재작성되고 데이터 베이스에 대한 하나의 요청으로 전송된다. 
    
    ```sql
    insert int author (age, genre, name, id) values (828, 'Genre_810', 'name', 810), (829, 'genre_811', 'name2', 811), ...
    ```
    
2. dataSource.hikari.cachePrepStmts: mysql은 preparedStatement caching을 비활성화 하고 있기 때문에, true를 지정해주어야 프리페어드 스테이트먼트에 캐싱을 줄 수 있다.
3. dataSource.hikari.prepStmtCacheSize(size) : 권장은 250~500. mysql 드라이버가 connection마다 캐싱할 preparedStatement의 개수를 지정하는 옵션
4. prepStmtCacheSqlLimit (preparedStatementsCacheSqlLimit) - 캐싱한 preparedStatement의 최대 길이를 지정하는 옵션. hibernate에선 기본값(256)이 턱없이 모자람. 권장은 2048
5. useServerPrepStmts: 서버측 프리페어드 스테이트먼트를 활성화 할 수 있다.

**mysql 클라이언트 프리페어드 스테이트먼트와 서버 프리페어드 스테이트먼트 부연설명**

1. 클라이언트 프리페어드 스테이트먼트: 클라이언트 프리페어드 스테이트먼트를 사용하면 sql문 실행을 위해 서버로 전송되기 전에 클라이언트 측에서 준비되는데, 위치 지정자(placeholder)를 실제 리터럴 값으로 대체해 sql문을 준비한다. 각 실행에서 클라이언트는 `COM_QUERY` 명령을 통해 실행할 준비가 된 완전한 sql문을 전송한다.
2. useServerPrepStmts=true를 설정하면 서버 프리페어드 스테이트먼트가 활성화되는데, 이번엔 sql 쿼리문이 `COM_STMT_PREPARE` 명령을 통해 클라이언트에서 서버로 한 번만 전송된다. 서버는 쿼리를 준비하고 결과를 클라이언트에 보낸다. 이후 각 실행에서 클라이언트는 `COM_STMT_EXCUTE` 명령으로 위치 지정자 대신 사용할 리터럴 값만 서버로 전송한다. 이 시점에 실제 SQL이 실행된다.
3. 대부분의 커넥션 풀은 커넥션 전체에서 프리페어드 스테이트먼트를 캐시한다. 즉, 동일한 명령문 문자열의 연속 호출은 동일한 PreparedStatement 인스턴스를 사용한다. 따라서 서버 측에서 동일한 문자열을 준비하지 않도록 동일한 PreparedStatement가 커넥션 전체에서 사용된다. 일부 커넥션 풀은 커넥션 풀 수준에서 프리페어드 스테이트먼트 캐시를 지원하지 않으며 JDBC 드라이버 캐싱(HikariCP)을 선호한다.
4. mysql 드라이버는 기본적으로 비활성화되는 클라이언트 측 명령문 캐시를 제공하는데, jdbc 옵션인 cachePrepStmts=true를 통해 활성화할 수 있다. 활성화되면 mysql은 클라이언트와 서버 프리페어드 스테이트먼트에 캐시를 제공한다. 다음 쿼리로 현재 캐싱 상태의 스냅숏을 얻을 수 있다. `SHOW GLOBAL STATUS LIKE '%stmt%';`

### 배치 등록을 위한 엔티티 준비 - IDENTITY 전략을 사용하면 bulk insert가 안된다.

JPA + Batch Insert를 사용하려면 ID 전략을 Table 전략으로 수정해야 하지만, **MySQL은 Sequence 전략이 없다.**

일반적으로 IDENTITY 전략은 id 획득을 db에게 위임을 한다.

- 따라서 Insert를 실행하기 전까지는 ID에 할당된 값을 미리 알 수 없기 때문에 Batch Insert를 진행할 수 없다.

Entity를 persist하려면 `@Id`로 지정한 필드에 값이 필요한데, **IDENTITY 타입은 실제 DB에 insert를 해야만 값을 얻을 수 있기 때문에 batch처리가 불가능하다.**

### 내장 saveAll(Iterable entities) 단점 확인 및 방지

1. 개발자는 현재 트랜잭션에서 영속성 컨텍스트 플러시 및 클리어를 제어할 수 없음: saveAll 메서드는 트랜잭션이 커밋되기 전에 단일 플러시를 발생시키며, **이로 인해 jdbc 배치를 준비하는 동안 엔티티는 현재 영속성 컨텍스트에 누적된다. 상당한 수의 엔티티의 경우 영속성 컨텍스트를 압도해 성능 저하(플러시가 느려짐) 또는 메모리 관련 오류가 유발될 수 있다.** 이를 막기 위해 일정한 크기로 saveAll를 호출하는 방법으로 해결한다.
2. 개발자는 merge() 대신 persist()를 사용할 수 없음: 내부적으로 saveAll 메서드는 EntityManager#merge()를 호출하는 내장 save(S s) 메서드를 호출한다. merge는 INSERT가 트리거되기 전에 JPA 영속성 공급자가 SELECT를 실행하여 동일한 기본키를 가진 레코드가 이미 디비에 포함돼 있지 않은지를 확인한다. 이런 SELECT가 많을 수록 성능 저하가 더 커진다. @Version 속성을 추가하면 배치 처리 전에 추가적인 SELECT를 막을 수 있다.
3. saveAll() 메서드는 영속된 엔티티를 포함하는 List를 반환함. 만약 리스트가 필요하지 않으면 의미없이 생성되는 것임. 가비지 컬렉터에 의미없는 작업만 추가로 시키게 되는 꼴임.

### 올바른 방법의 커스텀 구현

배치 처리의 커스텀 구현을 사용하면 프로세스를 제어하고 조정할 수 있다. 여러 최적화를 활용하는 saveInBatch() 메서드를 클라이언트에 제공하는데, 이 커스텀 구현은 EntityManager를 활용하며 다음과 같은 몇 가지 주요 목적을 갖는다.

1. 각 배치 처리 후 데이터베이스 트랜잭션 커밋
2. merge() 대신 persist() 사용
3. 추가 발생 SELECT를 피하고자 엔티티에 @Version이 필요치 않음
4. 영속된 엔티티에 대한 List를 반환하지 않음
5. saveInBatch(Iterable) 이름의 메서드를 통해 스프링 스타일 배치 처리 

```java
package com.bookstore.impl;

import org.springframework.data.jpa.repository.support.JpaEntityInformation;
import org.springframework.data.jpa.repository.support.SimpleJpaRepository;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import javax.persistence.EntityManager;
import java.io.Serializable;
import java.util.logging.Logger;

@Transactional(propagation = Propagation.NEVER)
public class BatchRepositoryImpl<T, ID extends Serializable>
        extends SimpleJpaRepository<T, ID> implements BatchRepository<T, ID> {
    private static final Logger logger = Logger.getLogger(BatchRepositoryImpl.class.getName());

    private final EntityManager entityManager;

    public BatchRepositoryImpl(JpaEntityInformation entityInformation, EntityManager entityManager) {
        super(entityInformation, entityManager);

        this.entityManager = entityManager;
    }

    @Override
    public <S extends T> void saveInBatch(Iterable<S> entities) {
        if (entities == null) {
            throw new IllegalArgumentException("The given Iterable of entities cannot be null!");
        }

        BatchExecutor batchExecutor = SpringContext.getBean(BatchExecutor.class);
        batchExecutor.saveInBatch(entities);
    }
}
```

```java
package com.bookstore.impl;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.persistence.EntityManager;
import javax.persistence.EntityManagerFactory;
import javax.persistence.EntityTransaction;
import java.util.logging.Level;
import java.util.logging.Logger;

@Component
public class BatchExecutor<T> {
    private static final Logger logger = Logger.getLogger(BatchExecutor.class.getName());

    @Value("${spring.jpa.properties.hibernate.jdbc.batch_size}")
    private int batchSize;

    private final EntityManagerFactory entityManagerFactory;

    public BatchExecutor(EntityManagerFactory entityManagerFactory) {
        this.entityManagerFactory = entityManagerFactory;
    }

    public <S extends T> void saveInBatch(Iterable<S> entities) {
        if (entities == null) {
            throw new IllegalArgumentException("The given Iterable of entities not be null!");
        }

        EntityManager entityManager = entityManagerFactory.createEntityManager();
        EntityTransaction entityTransaction = entityManager.getTransaction();

        try {
            entityTransaction.begin();

            int i = 0;
            for (S entity : entities) {
                if (i % batchSize == 0 && i > 0) {
                    logger.log(Level.INFO, "Flushing the EntityManager containing {0} entities ...", batchSize);

                    entityTransaction.commit();
                    entityTransaction.begin();

                    entityManager.clear();
                }

                entityManager.persist(entity);
                i++;
            }

            logger.log(Level.INFO, "Flushing the remaining entities ...");

            entityTransaction.commit();
        } catch (RuntimeException e) {
            if (entityTransaction.isActive()) {
                entityTransaction.rollback();
            }
            throw e;
        } finally {
            entityManager.close();
        }
    }
}
```

## 항목 47: 부모-자식 관계 배치 등록 최적화 방

저자가 40명, 도서가 저자당 5명씩 총 200권으로 총 240개의 Insert가 있고, batchSize가 15라고 가정하자. 등록 순서 지정(Ordering the inserts)없이는 insert into author, insert into book*5 이런 식으로 진행된다. jdbc 배치는 하나의 테이블 대상으로만 할 수 있어서 총 80번의 배치가 생성된다. 이를 최적화 하기 위해 배치 순서를 지정해야 한다.

### 배치 순서 지정

이 문제에 대한 해결 방법은 등록 순서 지정을 사용하는 것인데, application.properties에 추가해 처리한다.

`spring.jpa.properties.hibernate.order_inserts=true`

그렇게 되면

insert into author * 15 - 1회

insert into book * 15 * 5 - 5회

… 이런식으로 쭉 진행된다. 

책에 나온 실험결과는 Insert 개수가 2500인 케이스인 경우, 순서 지정 할때는 10초, 안 할때는 40초가 걸렸음.

## 항목 48. 세션 수준에서 배치 크기 제어 방법

애플리케이션 수준에서 배치 크기를 설정하는 것은 application.properties에 `spring.jpa.properties.hibernate.jdbc.batch_size` 를 지정해 처리한다. 그러나 하이버네이트 5.2부터 세션 수준에서 배치 크기를 설정할 수 있다. 이를 통해 배치 크기가 다른 하이버네이트 세션을 가질 수 있다.

```java
package com.bookstore.dao;

import org.hibernate.Session;

import org.springframework.stereotype.Repository;

import javax.persistence.EntityManager;
import javax.persistence.EntityManagerFactory;
import javax.persistence.EntityTransaction;
import java.io.Serializable;
import java.util.logging.Level;
import java.util.logging.Logger;

@Repository
public class Dao<T, ID extends Serializable> implements GenericDao<T, ID> {
    private static final Logger logger = Logger.getLogger(Dao.class.getName());

    private static final int BATCH_SIZE = 30;

    private final EntityManagerFactory entityManagerFactory;

    public Dao(EntityManagerFactory entityManagerFactory) {
        this.entityManagerFactory = entityManagerFactory;
    }

    @Override
    public <S extends T> void saveInBatch(Iterable<S> entities) {
        if (entities == null) {
            throw new IllegalArgumentException("The given Iterable of entities not be null!");
        }

        EntityManager entityManager = entityManagerFactory.createEntityManager();
        EntityTransaction entityTransaction = entityManager.getTransaction();

        Session session = entityManager.unwrap(Session.class);
        session.setJdbcBatchSize(BATCH_SIZE);

        try {
            entityTransaction.begin();

            int i = 0;
            for (S entity: entities) {
                if (i % session.getJdbcBatchSize() == 0 && i > 0) {
                    logger.log(Level.INFO, "Flushing the EntityManager containing {0} entities ...", session.getJdbcBatchSize());

                    entityTransaction.commit();
                    entityTransaction.begin();

                    entityManager.clear();
                }

                entityManager.persist(entity);
                i++;
            }

            logger.log(Level.INFO, "Flushing the remaining entities ...");

            entityTransaction.commit();
        } catch (RuntimeException e) {
            if (entityTransaction.isActive()) {
                entityTransaction.rollback();
            }
            throw e;
        } finally {
            entityManager.close();
        }
    }
}
```

## 항목 49. 포크/조인 JDBC 배치 처리 방법

jdbcTemplate을 통한 batch 처리방법은 PreparedStatement의 리터럴 값과 BatchPreparedStatementSetter의 인스턴스를 가지고 하나의 preparedStatement가 할당되고 setValues를 통해 업데이트가 된다. 이 방법은 매우 크지 않은 개수의 데이터에는 잘 동작하지만 큰 데이터인 경우 상당한 시간이 소요된다. 이에 대한 해결방법으로 책에서는 포크/조인 배치 처리를 제안한다.

### 포크/조인 배치 처리

배치 처리를 순차적으로 수행하는 대신 동시에 실행하는 것이 좋다. 자바에서는 Executors, 포크/조인 프레임 워크, CompletableFuture등과 같은 여러 방법이 있다. 이 중에서 포크/조인에 대해서 간단하게 알아보자.

포크/조인 프레임워크는 큰 작업을 수행하기 위한 것으로 병렬로 수행할 수 있는 더 작은 작업의 재귀적 분할(포크)를 위한 것이다. 최종적으로 모든 하위 작업이 완료된 후 해당 결과가 하나의 결과로 결합(조인) 된다.

포크/조인은 java.util.concurrent.ForkJoinPool을 통해 생성될 수 있다. ForkJoinPool 객체는 작업들을 다루는데, 그 안에서 실행되는 기본 타입은 ForkJoinTask<V>다. 작업에는 3가지 유형이 있지만 void를 반환하는 작업에 대한 RecursiveAction이 중요하다.

작업 처리 로직은 compute()라는 abstract 메서드에서 수행된다.

ForkJoinPool에 작업을 등록하는 것은 여러 메서드를 통해 가능하지만 invokeAll()이 중요하다. 이 메서드는 여러 작업 묶음을 포크하는데 사용된다. 

```java
package com.citylots.forkjoin;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveAction;

@Component
@Scope("prototype")
public class ForkingComponent extends RecursiveAction {
    @Value("${jdbc.batch.size}")
    private int batchSize;

    @Autowired
    private JoiningComponent joiningComponent;

    @Autowired
    private ApplicationContext applicationContext;

    private final List<String> jsonList;

    public ForkingComponent(List<String> jsonList) {
        this.jsonList = jsonList;
    }

    @Override
    public void compute() {
        if (jsonList.size() > batchSize) {
            ForkJoinTask.invokeAll(createSubtasks());
        } else {
            joiningComponent.executeBatch(jsonList);
        }
    }

    private List<ForkingComponent> createSubtasks() {
        List<ForkingComponent> subtasks = new ArrayList<>();

        int size = jsonList.size();

        List<String> jsonListOne = jsonList.subList(0, (size + 1) / 2);
        List<String> jsonListTwo = jsonList.subList((size + 1) / 2, size);

        subtasks.add(applicationContext.getBean(ForkingComponent.class, new ArrayList<>(jsonListOne)));
        subtasks.add(applicationContext.getBean(ForkingComponent.class, new ArrayList<>(jsonListTwo)));

        return subtasks;
    }
}
```

```java
package com.citylots.forkjoin;

import org.springframework.context.ApplicationContext;
import org.springframework.stereotype.Service;
import org.springframework.util.StopWatch;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.logging.Level;
import java.util.logging.Logger;

@Service
public class ForkJoinService {
    private final ApplicationContext applicationContext;

    public ForkJoinService(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    // ForkJoinPool will start 1 thread for each available core
    public static final int NUMBER_OF_LINES_TO_INSERT = 1000;
    public static final int NUMBER_OF_CORES = Runtime.getRuntime().availableProcessors();
    public static final ForkJoinPool forkJoinPool = new ForkJoinPool(NUMBER_OF_CORES);

    private static final Logger logger = Logger.getLogger(ForkJoinService.class.getName());

    public void fileToDatabase(String fileName) throws IOException {
        logger.info("Reading file lines into a List<String> ...");

        // fetch 200000+ lines
        List<String> allLines = Files.readAllLines(Path.of(fileName));

        // run on a snapshot of NUMBER_OF_LINES_TO_INSERT lines
        List<String> lines = allLines.subList(0, NUMBER_OF_LINES_TO_INSERT);

        logger.info(() -> "Read a total of " + allLines.size()
                + " lines, inserting " + NUMBER_OF_LINES_TO_INSERT + " lines");

        logger.info("Start inserting ...");

        StopWatch watch = new StopWatch();
        watch.start();
        forkjoin(lines);
        watch.stop();

        logger.log(Level.INFO, "Stop inserting. \n Total time: {0} ms ({1} s)",
                new Object[]{watch.getTotalTimeMillis(), watch.getTotalTimeSeconds()});
    }

    private void forkjoin(List<String> lines) {
        ForkingComponent forkingComponent = applicationContext.getBean(ForkingComponent.class, lines);
        forkJoinPool.invoke(forkingComponent);
    }
}
```

```java
package com.citylots.forkjoin;

import org.springframework.jdbc.core.BatchPreparedStatementSetter;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.List;

@Component
public class JoiningComponent {
    private static final String SQL_INSERT = "INSERT INTO lots (lot) VALUES (?)";

    private final JdbcTemplate jdbcTemplate;

    public JoiningComponent(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void executeBatch(List<String> jsonList) {
        jdbcTemplate.batchUpdate(SQL_INSERT,
            new BatchPreparedStatementSetter() {
                @Override
                public void setValues(PreparedStatement pStmt, int i) throws SQLException {
                    String jsonLine = jsonList.get(i);
                    pStmt.setString(1, jsonLine);
                }

                @Override
                public int getBatchSize() {
                    return jsonList.size();
                }
            });
    }
}
```

각 배치는 자체 트랜잭션/커넥션으로 실행되므로 포크/조인 스레드간 경합을 피하고자 커넥션 풀에 충분한 커넥션을 확보해놓아야 한다. 

실험결과는 25000개 기준으로 380초 걸리던 것이 8코어로 실행시 190초정도로 절반으로 줄었음.

## 항목 51. 배치 업데이트에 대한 효율적인 처리 방법

배치 업데이트도 항목 46과 같이 Mysql용 jdbc url이 필요하다. 다른 RDBMS일 경우 `spring.jpa.properties.hibernate.jdbc.batch_size` 를 통해 배치 크기를 설정한다.

업데이트 할때는 2가지 주요사항이 있다.

1. 버전이 지정된 엔티티
    
    업데이트돼야 할 엔티티에 버전이 지정된 경우 다음과 같이 설정이 구성돼 있는지 확인한다. `spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true`  참고로 하이버네이트5부터는 기본적으로 활성화된다.
    
2. 부모-자식 관계의 배치 업데이트
    
    업데이트가 전체/영속 전이와 함께 부모-자식 관계에 영향이 미치는 경우 다음 설정을 통해 업데이트 순서화하는게 좋다. `spring.jpa.properties.hibernate.order_update=true`
    

### 벌크 업데이트

벌크 작업은 빠르지만 다음과 같은 3가지 주요 단점을 갖고 있다.

1. 벌크 업데이트는 영속성 컨텍스트를 유효하지 않은 상태로 남길 수 있다.
2. 벌크 업데이트는 자동 낙관적 잠금의 이점을 얻지 못한다. @Version은 무시된다. 따라서 업데이트 손실이 방지 되지 않는다. 
3. 벌크 삭제는 전이되는 삭제(CascadeType.REMOVE) 또는 orphanRemoval의 장점을 이용할 수 없다.
