# 스프링 부트 JPA 모범사례

# 1장 연관관계

## 항목1: @OneToMany 연관관계를 효과적으로 구성하는 방법

이 @ManyToOne 연관관계는 외래키 열을 영속성 콘텍스트와 동기화 하는 역할을 한다.

예시로는 작가가 여러개의 책을 쓸 수 있으므로, author(부모, 참조될 키를 갖고 있는 엔티티), book(자식, 참조할 키를 갖고 있는 엔티티) 

### 항상 부모 측에서 자식측으로 전이를 사용

자식 측에서 부모 측으로의 전이는 코드 스멜이자 잘못된 관행. 자식이 부모의 탄생을 전이하는건 비논리적이지 않은가? Author에서 Book으로 전이될 수 있도록 Author 엔티티에 전이 타입을 추가한다.

### 부모 측에 mappedBy 지정 - 연관관계 주인 지정

부모 측에 설정되는 mappedBy 속성은 **양방향 연관관계의** 특성을 부여한다. 테이블 상에서는 외래키, 기본키 등으로 양방향 관계가 나타나지만, 객체 간에는 양방향이 아니라 단방향이 두개인 셈이다. 이때 객체의 두 관계 중 하나를 연관관계의 주인으로 지정을 하기 위해 mappedBy를 사용한다. 주인이 아닌 곳에 mappedBy를 지정하고 읽기만 가능하도록 설정.

### 부모 측에 orphanRemoval 지정 - 참조 없이 존재할 수 없는 의존 객체 정리

더이상 참조되지 않는 자식들의 삭제를 보장한다.

### 연관관계의 양측을 동기화 상태로 유지

부모 측 도우미 메서드를 통해 연관관계의 양쪽 상태를 동기화 할 수 있다. 연관관계의 양쪽을 동기화 상태로 유지하려 노력하지 않으면 엔티티 상태 전환으로 인해 예기치 않은 동작이 발생할 수 있다. 예를들면 book의 author를 다른 것으로 변경했지만 author의 books목록에서 제거가 같이되지 않으면, 데이터가 불일치하게됨.

### equals()와 hashCode() 오버라이딩

```java
@Test
void equalsTest() {
    EntityManager entityManager = entityManagerFactory.createEntityManager();
    EntityTransaction transaction = entityManager.getTransaction();
    transaction.begin();

    Item cake = Item.builder()
        .name("Cake")
        .price(2500)
        .stockQuantity(10).build();
    entityManager.persist(cake);
    transaction.commit();
    entityManager.clear();

    Item findCake = entityManager.find(Item.class, cake.getId());
    assertEquals(cake, findCake);
}
```

일반적으로는 한 엔티티 매니저의 영속성 컨텍스트에서 1차 캐시를 이용해 같은 ID의 엔티티를 항상 같은 객체로 가져올 수 있다. 하지만 위처럼 1차 캐시를 초기화한 후 다시 데이터베이스에서 동일한 엔티티를 읽어오는 경우 초기화 전에 얻었던 cake(2a984952)와 이후에 얻은 findCake(5366575d) 객체가 서로 다른 객체로 생성된 것을 볼 수 있다. 이를 방지하기 위해 오버라이딩이 필요하다.

### 연관관계 양측에서 지연로딩 사용해야 한다.

@OneToMany는 기본적으로 Lazy로딩이고, @ManyToOne은 eager 로딩인데, n+1 문제를 막기 위해서 명시적으로 Lazy를 지정해주어야 한다.

### toString() 오버라이딩 시 양방향 연관관계일 경우 무한 재귀 호출이 일어날 수 있음.

## 항목2: 단방향 @OneToMany 연관관계를 피해야 하는 이유

1. 누락된 관계를 나타내는 연결 테이블이 필요하여, 더 많은 메모리 사용
2. 이렇게 추가 연결테이블은 양방향에 비해 추가 쿼리가 필요
    1. 생성 시 연결 테이블  쿼리 추가됨
    2. 기존 저자의 새로운 등록에도 전체 삭제 후 다시 insert
    3. 마지막 도서 삭제시 전체 삭제 후 마지막 것을 제외한 나머지를 다시 Insert
3. 추가 쿼리 외에도 전체 삭제가 이루어지면 인덱스 항목 삭제 및 재추가로 인한 성능 저하도 있음

### @OrderColumn 사용

@OrderColumn 어노테이션을 지정하며 단방향 @OneToMany 연관관계가 정렬된다. @OrderColumn은 컬렉션이 ORDER BY 절을 사용해 정렬되도록 연결 테이블의 별도 데이터베이스 칼럼으로 항목 인덱스를 구체화 하도록 하이버네이트에 지시한다. 

근데 실제로는 잘 사용하진 않긴함.

### @JoinColumn 사용

@JoinColumn을 지정하면 @OneToMany 연관관계가 자식 테이블 외래키를 제어할 수 있음을 하이버네이트에 지시한다. 즉 연결 테이블이 필요 없어지게 된다.
