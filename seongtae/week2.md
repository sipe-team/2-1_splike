# 항목 131: 버전 기반(`@Version`) OptimisticLockException 발생 후 트랜잭션 재시도 방법

DB에 직접 락을 걸지 않고 동시성을 제어하는 기술

## 버전 기반 낙관적 잠금 예외

- `@Version`애노테이션을 선언한 필드를 엔티티에 추가해 버전을 관리
- JPA 영속성 관리자는 현재 트랜잭션이 다루는 데이터가 동시 트랜잭션에 의해 변경됐는지 확인할 수 있다. => 업데이트 손실이 되기 쉽다.

```
@Version의 타입은 short/Short가 좋다.
```

```java
@Entity
public class Inventory implements Serializable {

  private static final long serialVersionUID = 1L;

  @Id
  private Long id;

  private String title;
  private int quantity;

  @Version
  private Short version;

  public Short getVersion() {
    return version;
  }
}
```

두 개의 트랜잭션이 한 데이터에 관해 동시에 수행될 때, 엔티티에 `@Version` 이 있으면 시나리오 마지막 단계에서 OptimisticLockException이 발생한다.

재고와 관련된 요청임을 가정하여 최종적으로 한 개의 트랜잭션만 적용이 됐기에 비즈니스적으로 _1. 재고가 더 이상 없음을 클라이언트에 알리기_, _2. 재고가 충분할 경우 더 이상 재고가 없을 때까지 주문을 다시 시도_ 의 방법을 처리한다.

## 트랜잭션 재시도

별도의 db-util 과 같은 재시도 라이브러리로 트랜잭션을 다시 시도할 수 있다.

# 항목 132: 버전 없는 OptimisticLockException의 트랜잭션 재시도 방법

하이버네이트 ORM은 버전 없는 낙관적 락을 지원한다.

## 버전 없는 낙관적 잠금 예외

버전 없는 낙관적 잠금은 UPDATE 문에 추가되는 WHERE 절을 활용하는데, 이 절은 데이터가 현재 영속성 컨텍스트에서 가져온 이후 변경이 됐는지 확인한다.

```
현재의 영속성 콘텍스트가 열려있는 동안 작동하므로 엔티티 분리를 피해야 한다.
```

```java
@Entity
@DynamicUpdate
@OptimisticLocking(type = OptimisticLockType.DIRTY)
public class Inventory implements Serializable {

  private static final long serialVersionUID = 1L;

  @Id
  private Long id;

  private String title;
  private int quantity;
}
```

- OptimisticLockType.DIRTY를 설정하면 수정된 칼럼을 UPDATE WHERE 절에 자동으로 추가하게 하이버네이트에 지시
- `@DynamicUpdate` 애노테이션은 이 경우와 OptimisticLockType.ALL의 경우 필요
- `@OptimisticLock(excluded = true)` 애노테이션을 통해 필드 수준에서 특정 필드를 버전 관리에서 제외할 수 있다. => **OneToMany처럼 하위 컬렉션 변경 시이상위 버전 업데이트를 트리거하면 안된다.**

# 항목 133: 버전 기반 낙관적 잠금 및 분리된 엔터티를 처리하는 방법

- 하이버네이트 ORM의 버전 기반 낙관적 락은 분리된 엔티티가 작동한다.

예시)

- InventoryService에서 ID가 1인 Inventory 엔티티를 가져옴 - 트랜잭션A
- 다음 메서드는 동일한 ID의 Inventory 엔티티를 가져오고 업데이트 - 트랜잭션B
- 마지막 메서드는 트랜잭션 A에서 가져온 항목을 업데이트 - 트랜잭션C

1. 트랜잭션 A를 호출 - version 0
2. 트랜잭션 A 커밋 후 영속성 컨텍스트가 닫히며 Inventory는 분리된 엔티티가 됨.
3. 트랜잭션 B 호출 - version 1
4. 트랜잭션 B 커밋 후 영속성 컨텍스트가 닫히고 Inventory는 분리된 엔티티가 됨.
5. 트랜잭션 C 호출 <- 이전에 가져온 분리된 엔터티를 인수로 전달
6. 스프링은 엔티티를 검사하고 병합돼야 한다고 결정하여 내부적으로 `EntityManager#merge()`를 호출.
7. JPA 공급자는 DB에서 분리된 엔티티에 해당하는 영속 객체를 가져오고 분리된 엔티티를 영속된 객체로 복사.
8. 이 시점에 하이버네이트는 가져온 엔티티와 분리된 엔티티의 version이 일치하지 않는다고 결론을 내려 ObjectOptimisticLockingFailureException 예외 처리.

```
merge()를 사용하는 트랜잭션은 매번 버전이 분리된 엔티티의 버전과 일치하지 않는 엔티티를 데이터베이스에서 가져오므로 낙관적 잠금 예외가 항상 발생
```

# 항목 134: 장기 HTTP 통신에서의 낙관적 잠금 메커니즘 및 분리된 엔터티 사용 방법

- 논리적으로 관련된 여러 요청을 포함해 상태를 갖는 장기 통신을 구성한다.
- 조회 -> 수정 -> 쓰기 흐름은 여러 물리적 트랜잭션에 걸쳐 있는 논리적 또는 애플리케이션 수준 트랜잭션으로 인식된다.

```
애플리케이션 수준 트랜잭션은 ACID 특성에도 적합해야 한다.
장기 통신에서 데이터베이스에 변경 사항을 전파할 수 있는 것은 오직 마지막 물리적 트랜잭션에만 해당된다(flush, commit)
```

- 분리된 엔터티는 stateless HTTP 프로토콜을 통해 여러 요청에 걸쳐 있는 장기 통신에서 일반적으로 사용된다.

```
- HTTP 요청 A가 컨트롤러 엔드포인트에 도달
- 컨트롤러는 작업을 위임하고 영속성 콘텍스트 A에서 엔티티 A를 가져온다.
- 영속성 콘텍스트 A가 닫히고 엔터티 A는 분리된 상태가 된다.
- 분리된 엔티티 A는 세션에 저장되고 컨트롤러는 이를 클라이언트에 반환한다.
- 클라이언트는 수신된 데이터를 수정하고 다른 HTTP 요청 B에 수정 사항을 전송한다.
- 분리된 엔티티 A는 세션에서 가져와 클라이언트가 전송한 데이터와 동기화된다.
- 분리된 엔티티가 병합된다(하이버네이트는 데이터베이스의 최신 데이터 B를 영속성 컨텍스트 B에 로드하고 분리된 엔티티 A를 미러링하도록 업데이트한다).
- 병합 후 애플리케이션은 데이터베이스를 적절히 업데이트한다.
```

위 시나리오는 HTTP 요청 A와 B 사이에서 엔티티 데이터가 수정되지 않는 한 버전 기반 낙관적 잠금 없이도 잘 작동.

💡 중간에 데이터 변경 가능성이 있어서 업데이트 손실을 원하지 않는다면 버전 기반 낙관적 잠금이 필요하다.

```java
@Controller
@SessionAttributes({InventoryController.INVENTORY_ATTR})
public class InventoryController {
    protected static final String INVENTORY_ATTR = "inventory";
    private static final String BINDING_RESULT = "org.springframework.validation.BindingResult." + INVENTORY_ATTR;

    private final InventoryService inventoryService;

    public InventoryController(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @GetMapping("/load/{id}")
    public String fetchInventory(@PathVariable Long id, Model model) {
        if (!model.containsAttribute(BINDING_RESULT)) {
            model.addAttribute(INVENTORY_ATTR, inventoryService.fetchInventoryById(id));
        }

        return "index";
    }
    ...
```

위처럼 재고관련 컨트롤러가 존재한다고 가정한다.

이때, 새로운 수량을 지정하여 재고를 업데이트 할 때,

```java
    ...
    @PostMapping("/update")
    public String updateInventory(
            @Validated @ModelAttribute(INVENTORY_ATTR) Inventory inventory,
            BindingResult bindingResult, RedirectAttributes redirectAttributes,
            SessionStatus sessionStatus) {
        if (!bindingResult.hasErrors()) {
            try {
                Inventory updatedInventory =
                        inventoryService.updateInventory(inventory);
                redirectAttributes.addFlashAttribute("updatedInventory", updatedInventory);
            } catch (OptimisticLockingFailureException e) {
                bindingResult.reject("",
                        "Another user updated the data. Press the link above to reload it.");
            }
        }

        if (bindingResult.hasErrors()) {
            redirectAttributes.addFlashAttribute(BINDING_RESULT, bindingResult);
            return "redirect:load/" + inventory.getId();
        }

        sessionStatus.setComplete();

        return "redirect:success";
    }

    @GetMapping(value = "/success")
    public String success() {
        return "success";
    }

    @InitBinder
    void allowFields(WebDataBinder webDataBinder) {
        webDataBinder.setAllowedFields("quantity");
    }
}
```

@ModelAttribute와 @SessionAttribute의 역할

- 분리된 Inventory는 HTTP 세션에서 로드되고 전송된 데이터와 동기화한다.
- 분리된 Inventory는 업데이트돼 전송된 수정 사항을 반영한다.

updateInventory()

- 엔티티를 병합하고 수정 사항을 데이터베이스에 전파
- 그러는 사이 다른 수정 사항이 발생하면 낙관적 잠금 예외가 발생

# 엔티티가 수정되지 않은 경우에도 잠긴 엔티티 버전을 증가시키는 방법

- 다른 사람이 수정한 사항을 저장하지 않은 경우에만 변경 사항을 저장하고자 한다면? (= 순차적으로 수정 사항을 적용하고자 한다면?)
- Chapter와 Modification 엔티티가 서로 1:N 관계임을 가정

## OPTIMISTIC_FORCE_INCREMENT

- @Version과 OPTIMISTIC_FORCE_INCREMENT 잠금 전략 사용
- 이 두 가지 설정을 함께 사용하면 엔티티가 수정되지 않은 경우에도 잠긴 엔티티의 버전을 증가시킬 수 있다.
- 각각의 수정은 부모 엔티티의 낙관적 잠금 버전으로 강제 반영된다. (Chapter : Modification 관계에서 Modification 엔티티는 Chapter 엔티티의 버전을 따름)

```java
// Chapter.java
@Entity
public class Chapter implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
    private String content;

    @Version
    private short version;

    // ...
}
```

```java
// Modification.java
@Entity
public class Modification implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String description;
    private String modification;

    @ManyToOne(fetch = FetchType.LAZY)
    private Chapter chapter;

    // ...
}
```

`LockModeType.OPTIMISTIC_FORCE_INCREMENT` 잠금 전략을 사용해 ID로 챕터를 로드한다.

```java
@Repository
public interface ChapterRepository extends JpaRepository<Chapter, Long> {
    @Override
    @Lock(LockModeType.OPTIMISTIC_FORCE_INCREMENT)
    Optional<Chapter> findById(Long id);
}
```

이제 다음과 같은 시나리오를 가정한다.

1. 편집자 1이 챕터 1을 로드한다.
2. 편집자 2도 챕터 1을 로드한다.
3. 편집자 2는 수정을 반영하고 저장한다.
4. 편집자 2는 이 수정 사항을 챕터 1 낙관적 잠금 버전에 강제로 전파한다. 편집자 2의 트랜잭션이 커밋된다.
5. 편집자 1이 수정을 반영하고 저장하려고 시도한다.
6. 편집자 1은 그 사이에 편집자 2가 수정을 추가했기 때문에 낙관적 잠금 예외가 발생한다.

👍 => 자식 엔티티들이 부모 엔티티의 버전을 통해 낙관적 잠금 예외를 발생시킬 수 있다.

```
내부적으로 자식 엔티티가 수정됐을 때, 부모 엔티티의 낙관적 잠금 버전에 강제적으로 전파하기 때문!
```

## PESSIMISTIC_FORCE_INCREMENT

- OPTIMISTIC_FORCE_INCREMENT는 현재 트랝잭션이 끝날 때 버전 증가
- ↔️ PESSIMISTIC_FORCE_INCREMENT는 즉시 버전을 증가.
- 엔티티 버전은 행 수준 잠금(Write Lock)을 획득하기에 성공됨을 보장
- 증가는 엔티티가 데이터 액세스 계층으로 반환되기 전에 발생
- `@Lock(LockModeType.PESSIMISTIC_FORCE_INCREMENT)`를 사용

```java
@Repository
public interface ChapterRepository extends JpaRepository<Chapter, Long> {
    @Override
    @Lock(LockModeType.PESSIMISTIC_FORCE_INCREMENT)
    Optional<Chapter> findById(Long id);

    Chapter findByTitle(String title);
}
```

이제 다음과 같은 시나리오를 가정한다.

1. 편집자 1은 락을 획득하지 않고 챕터 1을 로드
2. 편집자 2도 챕터 1을 PESSIMISTIC_FORCE_INCREMENT로 로드
3. 편집자 2는 행 잠금을 얻고 버전을 즉시 증가
4. 편집자 2가 수정 사항을 저장
5. 편집자 1은 1. 에서 로드한 챕터 1 엔티티에 대해 PESSIMISTIC_FORCE_INCREMENT를 획득하려고 시도
6. 편집자 1은 낙관적 잠금 예외가 발생(편집자 2가 버전을 업데이트한 수정 사항을 추가했기 떄문).

```
배타적 락을 획득하고자 하이버네이트는 기본 Dialect 잠금 절을 사용
MySQL의 MySQL5Dialect(MyISAM)는 행 수준 잠금을 지원하지 않으며, MySQL5InnoDBDialect는 FOR UPDATE를 통해 행 수준 잠금을 획득.
MySQL8Dialect(InnoDB)는 FOR UPDATE NOWAIT를 통해 행 수준 잠금을 획득한다.

PostgreSQL에서 PostgreSQL95Dialect는 FOR UPDATE NOWAIT를 통해 행 수준 잠금을 획득

엔티티 버전을 증가시키는 트랜잭션은 행 수준 락은 해제할 때까지 PESSIMISTIC_FORCE_INCREMENT 락을 획득하려는 다른 트랜잭션을 차단.
-> 데드락을 피하고자 NOWAIT 또는 명시적인 짧은 타임아웃을 활용해야 한다.
```

```
MySQL은 REPEATABLE_READ를 기본 트랜잭션 격리 수준으로 사용하며, 획득한 락을 트랜잭션 기간 동안 유지함
READ_COMMITTED 격리 수준에서는 STATEMENT가 완료된 후 불필요한 잠금이 해제된다.
```
