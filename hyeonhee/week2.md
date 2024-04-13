## `@ElementCollection`, `@CollectionTable`

- String 등의 값 타입 또는 Embeddable 타입에 대한 단방향 일대다 연관관계를 간단히 정의할 때 사용
- 테이블 자동 생성
- 기본적으로 `@OneToMany`로 작동 → ~ToMany와 동일한 성능 저하를 가짐 → 기본 타입 또는 Embeddable에만 사용하자.
- 사용 시 `@OrderColumn` 함께 사용해서 INSERT DELETE 최적화 고려 필요 (toMany와 같은 이유)

```java
@Entity
public class ShoppingCart {
    // ...
    @ElementCollection(fetch = FetchType.LAZY)  // LAZY is default
    // @OrderColumn(name = "index_no")
    @CollectionTable(name = "shopping_cart_books",
            joinColumns = @JoinColumn(name = "shopping_cart_id"))
    private List<Book> books = new ArrayList<>();
}

@Embeddable
public class Book {...}
```

## 조회 방식 1 - fetch join

> \+ fetch join?
> - SQL에서 사용하는 조인의 종류는 아니고 JPQL에서 성능 최적화를 위해 제공하는 기능
> - 연관된 엔티티나 컬렉션을 한 번에 조회하는 기능

```java
@Repository
public interface ShoppingCartRepository extends JpaRepository<ShoppingCart, Long> {
    @Query(value = "SELECT p FROM ShoppingCart p JOIN FETCH p.books")
    ShoppingCart fetchShoppingCart();
}
```

## 조회 방식 2 - DTO 프로젝션

```java
@Repository
public interface ShoppingCartRepository extends JpaRepository<ShoppingCart, Long> {
    @Query(value = "SELECT a.owner AS owner, b.title AS title, b.price AS price "
            + "FROM ShoppingCart a JOIN a.books b")
    List<ShoppingCartDto> fetchShoppingCart();
}

public interface ShoppingCartDto {
    String getOwner();
    String getTitle();
    int getPrice();
}
```

### + `@Embedded`, `@Embeddable`

새로운 값 타입을 직접 정의해서 사용할 수 있는데, JPA에서는 이것을 임베디드 타입(embedded type)이라 합니다.

중요한 것은 직접 정의한 임베디드 타입도 `int`, `String`처럼 값 타입이라는 것입니다.

```java
// 임베디드 타입 사용
@Entity
public class Member {
  
  @Id @GeneratedVAlue
  private Long id;
  private String name;
  
  @Embedded
  private Period workPeriod;	// 근무 기간  
}

// 기간 임베디드 타입
@Embeddable
public class Peroid {
  
  @Temporal(TemporalType.DATE)
  Date startDate;
  @Temporal(TemporalType/Date)
  Date endDate;
  
  public boolean isWork (Date date) {
    // .. 값 타입을 위한 메서드를 정의할 수 있다
  }
}
```
