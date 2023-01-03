# 데이터 접근 기술 - MyBatis

## MyBatis 소개

- `MyBatis`는 `JdbcTemplate`보다 더 많은 기능을 제공하는 `SQL Mapper` 이다.
- `SQL`을 `XML`에 편리하게 작성할 수 있고 또 동적 쿼리를 매우 편리하게 작성할 수 있다

## MyBatis 설정

```java
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'

#MyBatis application.properties
mybatis.type-aliases-package=hello.itemservice.domain
mybatis.configuration.map-underscore-to-camel-case=true
logging.level.hello.itemservice.repository.mybatis=trace
```

- `mybatis.type-aliases-package`
    - 마이바티스에서 타입 정보를 사용할 때는 패키지 이름을 적어주어야 하는데, 여기에 명시하면 패키지 이름을 생략할 수 있다.
    - 지정한 패키지와 그 하위 패키지가 자동으로 인식된다.
    - 여러 위치를 지정하려면 `,` , `;` 로 구분하면 된다.
- `mybatis.configuration.map-underscore-to-camel-case`
    - `JdbcTemplate`의 `BeanPropertyRowMapper` 에서 처럼 언더바를 카멜로 자동 변경해주는 기능을 활성화 한다.
- `logging.level.hello.itemservice.repository.mybatis=trace`
    - `MyBatis`에서 실행되는 쿼리 로그를 확인할 수 있다

## MyBatis 적용

```java
@Mapper
public interface ItemMapper {

    void save(Item item);

    void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto updateParam);

    List<Item> findAll(ItemSearchCond itemSearchCond);
    
    Optional<Item> findById(Long id);
}
```

- 마이바티스 매핑 `XML`을 호출해주는 매퍼 인터페이스이다.
- 이 인터페이스에는 `@Mapper` 애노테이션을 붙여주어야 한다. 그래야 `MyBatis`에서 인식할 수 있다.
- 이 인터페이스의 메서드를 호출하면 다음에 보이는 `xml` 의 해당 `SQL`을 실행하고 결과를 돌려준다
- **xml 적용**
    - `hello.itemservice.repository.mybatis.ItemMapper`
        - 자바 코드가 아니기 때문에 `src/main/resources` 하위에 만들되, 패키지 위치는 맞추어 주어야 한다.
    - `mybatis.mapper-locations=classpath:mapper/**/*.xml`
        - `resources/mapper` 를 포함한 그 하위 폴더에 있는 `XML`을 `XML 매핑 파일`로 인식한다. 이 경우 파일 이름은 자유롭게 설정해도 된다.

```java
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">

    <insert id="save" useGeneratedKeys="true" keyProperty="id">
        insert into item (item_name, price, quantity)
        values (#{itemName}, #{price}, #{quantity})
    </insert>

    <update id="update">
        update item
        set item_name = #{updateParam.itemName},
            price = #{updateParam.price},
            quantity=#{updateParam.quantity}
        where id = #{id}
    </update>

    <select id="findById" resultType="Item">
        select *
        form item
        where id = #{id}
    </select>

    <select id="findAll">
        select id, item_name, price, quantity
        from item
        <where>
            <if test="itemName != null and itemName != ''">
                and item_name like concat('%', #{itemName}, '%')
            </if>
            <if test="maxPrice != null">
                and price &lt;= #{maxPrice}
            </if>
        </where>
    </select>

</mapper>
```

- **save**
    - `id` 에는 매퍼 인터페이스에 설정한 메서드 이름을 지정하면 된다. 여기서는 메서드 이름이 `save()` 이므로 `save` 로 지정하면 된다.
    - 파라미터는 `#{}` 문법을 사용하면 된다. 그리고 매퍼에서 넘긴 객체의 프로퍼티 이름을 적어주면 된다.
    - `#{}` 문법을 사용하면 `PreparedStatement` 를 사용한다. `JDBC`의 `?` 를 치환한다 생각하면 된다.
    - `useGeneratedKeys` 는 데이터베이스가 키를 생성해 주는 `IDENTITY` 전략일 때 사용한다. `keyProperty` 는 생성되는 키의 속성 이름을 지정한다. `Insert`가 끝나면 `item` 객체의 `id` 속성에 생성된 값이 입력된다
- **update**
    - 파라미터가 `Long id` , `ItemUpdateDto updateParam` 으로 2개이다. 파라미터가 1개만 있으면 `@Param` 을 지정하지 않아도 되지만, 파라미터가 2개 이상이면 `@Param` 으로 이름을 지정해서 파라미터를 구분해야 한다
- **findById**
    - `resultType` 은 반환 타입을 명시하면 된다.
        - `mybatis.type-aliasespackage=hello.itemservice.domain` 속성을 지정한 덕분에 모든 패키지 명을 다 적지는 않아도 된다.
        - `JdbcTemplate`의 `BeanPropertyRowMapper` 처럼 `SELECT SQL`의 결과를 편리하게 객체로 바로 변환해준다
    - 자바 코드에서 반환 객체가 하나이면 `Item` , `Optional<Item>` 과 같이 사용하면 되고, 반환 객체가 하나 이상이면 컬렉션을 사용하면 된다. 주로 `List` 를 사용한다.
- **findAll**
    - `<if>` 는 해당 조건이 만족하면 구문을 추가한다.
    - `<where>` 은 적절하게 `where` 문장을 만들어준다.
        - 예제에서 `<if>` 가 모두 실패하게 되면 `SQL where` 를 만들지 않는다.
        - 예제에서 `<if>` 가 하나라도 성공하면 처음 나타나는 `and` 를 `where` 로 변환해준다.

```java
@Repository
@RequiredArgsConstructor
public class MyBatisItemRepository implements ItemRepository {

    private final ItemMapper itemMapper;

    @Override
    public Item save(Item item) {
        itemMapper.save(item);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        itemMapper.update(itemId,updateParam);
    }

    @Override
    public Optional<Item> findById(Long id) {
        return itemMapper.findById(id);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        return itemMapper.findAll(cond);
    }
}

@Configuration
@RequiredArgsConstructor
public class MyBatisConfig {

    private final ItemMapper itemMapper;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
    @Bean
    public ItemRepository itemRepository() {
        return new MyBatisItemRepository(itemMapper);
    }
}
```

## MyBatis 분석

![image](https://user-images.githubusercontent.com/105543967/210307564-2fe66682-4a24-4209-b75b-b671354eb62d.png)


1. 애플리케이션 로딩 시점에 `MyBatis` 스프링 연동 모듈은 `@Mapper` 가 붙어있는 인터페이스를 조사한다.
2. 해당 인터페이스가 발견되면 동적 프록시 기술을 사용해서 `ItemMapper` 인터페이스의 구현체를 만든다.
3. 생성된 구현체를 스프링 빈으로 등록한다
- **매퍼 구현체**
    - 마이바티스 스프링 연동 모듈이 만들어주는 `ItemMapper` 의 구현체 덕분에 인터페이스 만으로 편리하게 `XML`의 데이터를 찾아서 호출할 수 있다.
    - 원래 마이바티스를 사용하려면 더 번잡한 코드를 거쳐야 하는데, 이런 부분을 인터페이스 하나로 매우 깔끔하고 편리하게 사용할 수 있다.
    - 매퍼 구현체는 예외 변환까지 처리해준다. `MyBatis`에서 발생한 예외를 스프링 예외 추상화인 `DataAccessException` 에 맞게 변환해서 반환해준다.

## MyBatis 기능 - 동적 쿼리

- 동적 SQL
    - 마이바티스가 제공하는 최고의 기능이자 마이바티스를 사용하는 이유는 바로 동적 SQL 기능 때문이다. 동적 쿼리를 위해 제공되는 기능은 다음과 같다.
    - `if`
        - 내부의 문법은 OGML을 사용한다.
    - `choose (when, otherwise)`
        - 자바의 switch 구문과 유사한 구문도 사용 가능.
    - `trim (where, set)`
    - `foreach`
        - 컬렉션을 반복 처리할 때 사용한다. `where in (1,2,3,4,5,6)` 와 같은 문장을 쉽게 완성할 수 있다. 파라미터로 `List` 를 전달하면 된다.

## MyBatis 기능 - 기타 기능

- **애노테이션으로 SQL 작성**
    - `@Insert`, `@Update`, `@Delete`, `@Select` 기능이 제공된다.

```java
@Select("select id, item_name, price, quantity from item where id=#{id}")
Optional<Item> findById(Long id);
```

- **문자열 대체(String Substitution)**
    - `#{}` 문법은 `?`를 넣고 파라미터를 바인딩하는 `PreparedStatement` 를 사용한다.
- **재사용 가능한 SQL 조각**
    - `<sql>` 을 사용하면 `SQL` 코드를 재사용 할 수 있다

`<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>`

- **Result Maps**
    - 별칭을 사용하지 않고도 문제를 해결할 수 있는데, `resultMap` 을 선언해서 사용하면 된다.

```java
<resultMap id="userResultMap" type="User">
 <id property="id" column="user_id" />
 <result property="username" column="username"/>
 <result property="password" column="password"/>
</resultMap>
<select id="selectUsers" resultMap="userResultMap">
 select user_id, user_name, hashed_password
 from some_table
 where id = #{id}
</select>
```

- MyBatis는 앞서 설명한 JdbcTemplate보다 더 많은 기능을 제공하는 SQL Mapper 이다.
기본적으로 JdbcTemplate이 제공하는 대부분의 기능을 제공한다.
- JdbcTemplate과 비교해서 MyBatis의 가장 매력적인 점은 SQL을 XML에 편리하게 작성할 수 있고 또 **동적 쿼리를 매우 편리하게 작성할 수 있다**는 점이다.
- 다만 JdbcTemplate은 스프링에 내장된 기능이여서 별도의 설정없이 사용가능하지만, MyBatis는 약간의 설정이 필요하다.
