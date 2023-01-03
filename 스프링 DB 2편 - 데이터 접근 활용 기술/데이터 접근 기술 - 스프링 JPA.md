# 데이터 접근 기술 - JPA

## ORM(Object-relational mapping(객체 관계 매핑))

- 객체는 객체대로 설계
- 관계형 데이터베이스는 관계형 데이터베이스대로 설계
- `ORM` 프레임워크가 중간에서 매핑
- 대중적인 언어에는 대부분 `ORM` 기술이 존재
- `JPA`는 애플리케이션과 `JDBC` 사이에서 동작

## JPA 소개

- `EJB - 엔티티 빈` → `하이버네이트(오픈 소스)` → `JPA(자바 표준)`
- `JPA`는 인터페이스의 모음
- **JPA 사용 이유**
    - `SQL` 중심적인 개발에서 `객체` 중심적인 개발
    - `생산성`, `유지보수`, `성능`. `표준`
    - 패러다임의 불일치 해결
    - 데이터 접근 추상화 벤더 독립성
- **JPA와 패러다임의 불일치 해결**
    - JPA와 상속
    - JPA와 연관관계
    - JPA와 객체 그래프 탐색
    - JPA와 비교하기
- **JPA의 성능 최적화 기능**
    - 1차 캐시와 동일성(identity) 보장
        - 같은 트랜잭션 안에서 같은 엔티티 반환 - 조회 성능 향상
    - 트랜잭션을 지원하는 쓰기 지연(transactional write-behind)
        - 트랜잭션을 커밋할 때가지 `INSERT SQL`을 모음
        - `JDBC BATCH SQL` 기능을 사용해서 한번에 `SQL` 전송
    - 지연 로딩(Lazy Loading)
        - 지연 로딩 : 객체가 실제 사용될 때 로딩
        - 즉시 로딩 : `JOIN SQL`로 한번에 연관된 객체까지 미리 조회

```
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE

```

## JPA 적용

```java
@Data
@Entity
// @Table(name = "item")
public class Item {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "item_name", length = 10)
    private String itemName;
    private Integer price;
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}

```

- `@Entity` : `JPA`가 사용하는 객체라는 뜻이다. 이 에노테이션이 있어야 `JPA`가 인식할 수 있다. 이렇게 `@Entity` 가 붙은 객체를 `JPA`에서는 엔티티라 한다.
- `@Id` : 테이블의 PK와 해당 필드를 매핑한다.
- `@GeneratedValue(strategy = GenerationType.IDENTITY)` : PK 생성 값을 데이터베이스에서 생성하는 `IDENTITY` 방식을 사용한다.
- `@Column` : 객체의 필드를 테이블의 컬럼과 매핑한다.
- JPA는 `public`또는 `protected`의 기본 생성자가 필수이다.

```java
@Slf4j
@Repository
@Transactional
public class JpaItemRepository implements ItemRepository {

    private final EntityManager em;

    public JpaItemRepository(EntityManager em) {
        this.em = em;
    }

    @Override
    public Item save(Item item) {
        em.persist(item);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = em.find(Item.class, itemId);
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }

    @Override
    public Optional<Item> findById(Long id) {
        Item item = em.find(Item.class, id);
        return Optional.ofNullable(item);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String jpql = "select i from Item i";

        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();

        if (StringUtils.hasText(itemName) || maxPrice != null) {
            jpql += " where";
        }
        boolean andFlag = false;
        List<Object> param = new ArrayList<>();
        if (StringUtils.hasText(itemName)) {
            jpql += " i.itemName like concat('%',:itemName,'%')";
            param.add(itemName);
            andFlag = true;
        }
        if (maxPrice != null) {
            if (andFlag) {
                jpql += " and";
            }
            jpql += " i.price <= :maxPrice";
            param.add(maxPrice);
        }
        log.info("jpql={}", jpql);
        TypedQuery<Item> query = em.createQuery(jpql, Item.class);
        if (StringUtils.hasText(itemName)) {
            query.setParameter("itemName", itemName);
        }
        if (maxPrice != null) {
            query.setParameter("maxPrice", maxPrice);
        }
        return query.getResultList();
    }
}

```

- `private final EntityManager em` : 생성자를 보면 스프링을 통해 엔티티 매니저 `(EntityManager)` 라는 것을 주입받은 것을 확인할 수 있다. `JPA`의 모든 동작은 엔티티 매니저를 통해서 이루어진다. 엔티티 매니저는 내부에 데이터소스를 가지고 있고, 데이터베이스에 접근할 수 있다.
- `@Transactional` : `JPA`의 모든 데이터 변경(등록, 수정, 삭제)은 트랜잭션 안에서 이루어져야 한다. 조회는 트랜잭션이 없어도 가능하다. 변경의 경우 일반적으로 서비스 계층에서 트랜잭션을 시작하기 때문에 문제가 없다. `JPA`에서는 데이터 변경시 트랜잭션이 필수다.
- **save()**
    - `IDENTITY` 로 사용했기 때문에 `JPA`가 이런 쿼리를 만들어서 실행한 것이다. 물론 쿼리 실행 이후에 `Item` 객체의 `id` 필드에 데이터베이스가 생성한 `PK값`이 들어가게 된다.
- **update() - 수정**
    - `JPA`은 트랜잭션 커밋 시점에 `JPA`가 변경된 엔티티 객체를 찾아서 `UPDATE SQL`을 수행
- **findById()**
    - `JPA`에서 엔티티 객체를 `PK`를 기준으로 조회할 때는 `find`() 를 사용하고 조회 타입과, `PK` 값을 주면 된다.
- **findAll()**
    - **JPQL**
        - `JPA`는 `JPQL(Java Persistence Query Language)`이라는 객체지향 쿼리 언어를 제공한다.
        - 여러 데이터를 복잡한 조건으로 조회할 때 사용한다

## JPA 예외 변환

- `EntityManager` 는 순수한 `JPA` 기술이고, 스프링과는 관계가 없다. 따라서 엔티티 매니저는 예외가 발생하면 `JPA` 관련 예외를 발생시킨다.
- `JPA`는 `PersistenceException` 과 그 하위 예외를 발생시킨다
    - `IllegalStateException` , `IllegalArgumentException` 을 발생시킬 수 있다
- **@Repository의 기능**
    - `@Repository`가 붙은 클래스는 컴포넌트 스캔 대상이 된다.
    - `@Repository` 가 붙은 클래스는 예외 변환 AOP의 적용 대상이 된다.
        - 스프링과 JPA를 함께 사용하는 경우 스프링은 JPA 예외 변환기 `(PersistenceExceptionTranslator)`를 등록한다.
        - 예외 변환 `AOP 프록시`는 `JPA 관련 예외`가 발생하면 `JPA 예외 변환기`를 통해 발생한 예외를 스프링 데이터 접근 예외로 변환한다
