# 데이터 접근기술 - 스프링 데이터 JPA

## 스프링 데이터 JPA 기능

- 스프링 데이터 JPA는 JPA를 편리하게 사용할 수 있도록 도와주는 라이브러리이다.
- **스프링 데이터 JPA의 대표적 기능**
    - 공통 인터페이스 기능
    - 쿼리 메서드 기능
- **공통 인터페이스 기능**

![https://user-images.githubusercontent.com/106054507/190154745-1f37328a-eee8-4442-86f9-4577886f5294.png](https://user-images.githubusercontent.com/106054507/190154745-1f37328a-eee8-4442-86f9-4577886f5294.png)

- `JpaRepository` 인터페이스를 통해서 기본적인 `CRUD` 기능 제공한다.
- 공통화 가능한 기능이 거의 모두 포함되어 있다.
- `CrudRepository` 에서 `fineOne()` → `findById()` 로 변경되었다.
- **쿼리메서드 기능**
    - 스프링 데이터 JPA는 인터페이스에 메서드만 적어두면, 메서드 이름을 분석해서 쿼리를 자동으로 만들고 실행해주는 기능을 제공

```
public interface MemberRepository extends JpaRepository<Member, Long> {
 List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}

```

- **스프링 데이터 JPA가 제공하는 쿼리 메소드 기능**
    - 조회: `find…By` , `read…By` , `query…By` , `get…By`
    - COUNT: `count…By` 반환타입 `long`
    - EXISTS: `exists…By` 반환타입 `boolean`
    - 삭제: `delete…By` , `remove…By` 반환타입 `long`
    - DISTINCT: `findDistinct` , `findMemberDistinctBy`
    - LIMIT: `findFirst3` , `findFirst` , `findTop` , `findTop3`
- **JPQL 사용**
    - `JPQL`을 사용하고 싶을 때는 `@Query` 와 함께 `JPQL`을 작성하면 된다. 이때는 메서드 이름으로 실행하는 규칙은 무시된다.
    - `JPQL` 뿐만 아니라 `JPA`의 네이티브 쿼리 기능도 지원한다..

## 스프링 데이터 JPA 적용

```java
public interface SpringDataJpaItemRepository extends JpaRepository<Item, Long> {

    List<Item> findByNameLike(String itemName);

    List<Item> findByPriceLessThanEqual(Integer price);

    //쿼리 메서드 (아래 메서드와 같은 기능 수행)
    List<Item> findByItemNameLikeAndPriceLessThanEqual(String itemName, Integer price);

    //쿼리 직접 실행
    @Query("select i from Item i where i.itemName like :itemName and i.price <= :price")
    List<Item> findItems(@Param("itemName") String itemName, @Param("price") Integer price);
}

@Repository
@RequiredArgsConstructor
public class JpaItemRepositoryV2 implements ItemRepository {

    private final SpringDataJpaItemRepository repository;

    @Override
    public Item save(Item item) {
        return repository.save(item);
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = repository.findById(itemId).orElseThrow();
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }

    @Override
    public Optional<Item> findById(Long id) {
        return repository.findById(id);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        if (StringUtils.hasText(itemName) && maxPrice != null) {
//            return repository.findByItemNameLikeAndPriceLessThanEqual(itemName, maxPrice);
            return repository.findItems(itemName,maxPrice);
        } else if (StringUtils.hasText(itemName)) {
            return repository.findByNameLike(itemName);
        } else if (maxPrice != null) {
            return repository.findByPriceLessThanEqual(maxPrice);
        }else{
            return repository.findAll();
        }
    }
}

```

- `findByItemNameLikeAndPriceLessThanEqual()` 를 사용해도 되고,
`repository.findItems()` 를 사용해도 된다. 그런데 보는 것 처럼 조건이 2개만 되어도 이름이 너무 길어지는 단점이 있다.
- **예외 변환**
- 스프링 데이터 `JPA`도 스프링 예외 추상화를 지원한다. 스프링 데이터 `JPA`가 만들어주는 프록시에서 이미 예외 변환을 처리하기 때문에, `@Repository` 와 관계없이 예외가 변환된다.

## QueryDsl

- `JPA`, `MongoDB`, `SQL` 같은 기술들을 위해 `type-safe SQL`을 만드는 프레임워크
- 동작방식 : `QueryDsl` → `JPQL` → `SQL`
- **QueryDsl 설정**

```java
//Querydsl 추가
implementation 'com.querydsl:querydsl-jpa'
annotationProcessor "com.querydsl:querydsl-apt:$
{dependencyManagement.importedProperties['querydsl.version']}:jpa"
annotationProcessor "jakarta.annotation:jakarta.annotation-api"
annotationProcessor "jakarta.persistence:jakarta.persistence-api"

//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
clean {
delete file('src/main/generated')
}

```

- `Preferences` → `Build, Execution, Deployment` → `Build Tools` → `Gradle`
- **Gradle IntelliJ 사용법**
    - `Gradle` → `Tasks` → `build` → `clean`
    - `Gradle` → `Tasks` → `other` → `compileJava`
- **Gradle 콘솔 사용법**
    - `./gradlew clean compileJava`

```java
@Repository
@Transactional
public class JpaItemRepositoryV3 implements ItemRepository {

    private final EntityManager em;
    private final JPAQueryFactory query;

    public JpaItemRepositoryV3(EntityManager em) {
        this.em = em;
        this.query = new JPAQueryFactory(em);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        BooleanBuilder builder = new BooleanBuilder();
        if (StringUtils.hasText(itemName)) {
            builder.and(item.itemName.like("%" + itemName + "%"));
        }
        if (maxPrice != null) {
            builder.and(item.price.loe(maxPrice));
        }

        List<Item> result = query.select(item)
                .from(item)
                .where()
                .fetch();

        return result;
    }

		@Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        return query.select(item)
                .from(item)
                .where(likeItemName(itemName), maxPrice(maxPrice))
                .fetch();
    }

    private BooleanExpression maxPrice(Integer maxPrice) {
        if (maxPrice != null) {
            return item.price.loe(maxPrice);
        }
        return null;
    }

    private BooleanExpression likeItemName(String itemName) {
        if (StringUtils.hasText(itemName)) {
            return item.itemName.like("%" + itemName + "%");
        }
        return null;
    }
}

```

- `Querydsl`에서 `where(A,B)` 에 다양한 조건들을 직접 넣을 수 있는데, 이렇게 넣으면 `AND` 조건으로 처리된다. 참고로 `where()` 에 `null` 을 입력하면 해당 조건은 무시한다.
- 다른 장점은 `likeItemName()` , `maxPrice()` 를 다른 쿼리를 작성할 때 재사용 할 수 있다는 점이다. 쉽게 이야기해서 쿼리 조건을 부분적으로 모듈화 할 수 있다

## 트레이드 오프

- `DI`, `OCP`를 지키기 위해 어댑터를 도입하고, 더 많은 코드를 유지한다.
- 어댑터를 제거하고 구조를 단순하게 가져가지만, `DI`, `OCP`를 포기하고, `ItemService` 코드를 직접 변경한다.

## 플러시 타이밍

- `JPA`와 `JdbcTemplate`을 함께 사용할 경우 `JPA`의 플러시 타이밍에 주의해야 한다.
- `JPA`는 데이터를 변경하면 변경 사항을 즉시 데이터베이스에 반영하지 않는다. 기본적으로 트랜잭션이 커밋되는 시점에 변경 사항을 데이터베이스에 반영한다. 그래서 하나의 트랜잭션 안에서 `JPA`를 통해 데이터를 변경한 다음에 `JdbcTemplate`을 호출하는 경우 `JdbcTemplate`에서는 `JPA`가 변경한 데이터를 읽지 못하는 문제가 발생한다.
- 이 문제를 해결하려면 `JPA` 호출이 끝난 시점에 `JPA`가 제공하는 플러시라는 기능을 사용해서 `JPA`의 변경 내역을 데이터베이스에 반영해주어야 한다. 그래야 그 다음에 호출되는 `JdbcTemplate`에서 `JPA`가 반영한 데이터를 사용할 수 있다
