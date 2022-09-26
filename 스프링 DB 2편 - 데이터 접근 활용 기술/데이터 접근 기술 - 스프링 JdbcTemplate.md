# 데이터 접근 기술 - 스프링 JdbcTemplate

## JdbcTemplate 소개

**장점**

- 설정의 편리함
    - JdbcTemplate은 spring-jdbc 라이브러리에 포함되어 있는데, 이 라이브러리는 스프링으로 JDBC를 사용할 때 기본으로 사용되는 라이브러리이다. 그리고 별도의 복잡한 설정 없이 바로 사용할 수 있다.
- 반복 문제 해결
    - JdbcTemplate은 템플릿 콜백 패턴을 사용해서, JDBC를 직접 사용할 때 발생하는 대부분의 반복 작업을 대신 처리해준다.
    - 개발자는 SQL을 작성하고, 전달할 파리미터를 정의하고, 응답 값을 매핑하기만 하면 된다.
    - 우리가 생각할 수 있는 대부분의 반복 작업을 대신 처리해준다.
        - 커넥션 획득
        - statement 를 준비하고 실행
        - 결과를 반복하도록 루프를 실행
        - 커넥션 종료, statement , resultset 종료
        - 트랜잭션 다루기 위한 커넥션 동기화
        - 예외 발생시 스프링 예외 변환기 실행

**단점**

- 동적 SQL을 해결하기 어렵다

## JdbcTemplate 이름지정 파라미터

**순서대로 바인딩**

```java
String sql = "update item set item_name=?, price=?, quantity=? where id=?";
template.update(sql,
 itemName,
 price,
 quantity,
 itemId);
```

여기서는 `itemName` , `price` , `quantity` 가 SQL에 있는 ? 에 순서대로 바인딩 된다 하지만

```java
String sql = "update item set item_name=?, quantity=?, price=? where id=?";
template.update(sql,
 itemName,
 price,
 quantity,
 itemId);
```

이렇게 되면 다음과 같은 순서로 데이터가 바인딩 된다

`item_name=itemName, quantity=price, price=quantity`

**이름 지정 바인딩**

```java
insert into item (item_name, price, quantity) " +
 "values (:itemName, :price, :quantity)"
```

파라미터를 전달하려면 `Map` 처럼 `key` , `value` 데이터 구조를 만들어서 전달해야 한다. 여기서 `key` 는 :파리이터이름 으로 지정한, 파라미터의 이름이고 , `value` 는 해당 파라미터의 값이 된다

이름 지정 바인딩에서 자주 사용하는 파라미터의 종류는 크게 3가지가 있다.

- Map
    - 단순히 Map 을 사용한다
    
    ```java
    Map<String, Object> param = Map.of("id", id);
    Item item = template.queryForObject(sql, param, itemRowMapper());
    ```
    
- SqlParameterSource
    - MapSqlParameterSource
        - `Map` 과 유사한데, SQL 타입을 지정할 수 있는 등 SQL에 좀 더 특화된 기능을 제공한다.
        - `SqlParameterSource` 인터페이스의 구현체이다.
        - `MapSqlParameterSource` 는 메서드 체인을 통해 편리한 사용법도 제공한다
        
        ```java
        SqlParameterSource param = new MapSqlParameterSource()
         .addValue("itemName", updateParam.getItemName())
         .addValue("price", updateParam.getPrice())
         .addValue("quantity", updateParam.getQuantity())
         .addValue("id", itemId); //이 부분이 별도로 필요하다.
        template.update(sql, param);
        ```
        
    - BeanPropertySqlParameterSource
        - 자바빈 프로퍼티 규약을 통해서 자동으로 파라미터 객체를 생성한다.
        
        ```idris
        SqlParameterSource param = new BeanPropertySqlParameterSource(item);
        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(sql, param, keyHolder);
        ```
        

**별칭**

`select item_name as itemName`

별칭 `as`를 사용해서 SQL 조회 결과의 이름을 변경하는 것이다.

**관계의 불일치**

자바 객체는 카멜( `camelCase` ) 표기법을 사용한다. 반면에 관계형 데이터베이스에서는 주로 언더스코어를 사용하는 `snake_case` 표기법을 사용한다
