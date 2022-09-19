# 검증2 - Bean Validation

## Bean Validation - 소개

검증 기능을 지금처럼 매번 코드로 작성하는 것은 상당히 번거롭다. 특히 특정 필드에 대한 검증 로직은 대부분 빈 값인지 아닌지, 특정 크기를 넘는지 아닌지와 같이 매우 일반적인 로직이다.

```java
public class Item {

  private Long id;
  
  @NotBlank
  private String itemName;

  @NotNull
  @Range(min = 1000, max = 1000000)
  private Integer price;

  @NotNull
  @Max(9999)
  private Integer quantity;
 //...
}
```

### Bean Validation 이란?

먼저 Bean Validation은 특정한 구현체가 아니라 Bean Validation 2.0(JSR-380)이라는 기술 표준이다. 쉽게 이야기해서 검증 애노테이션과 여러 인터페이스의 모음이다.

**하이버네이트 Validator 관련 링크**

- 공식 사이트: [http://hibernate.org/validator/](http://hibernate.org/validator/)
- 공식 메뉴얼: [https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/)
- 검증 애노테이션 모음: [https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/#validator-defineconstraints-spec](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/#validator-defineconstraints-spec)

## Bean Validation - 시작

### **Bean Validation 의존관계 추가**

**의존관계 추가**

Bean Validation을 사용하려면 다음 의존관계를 추가해야 한다.

`build.gradle`

`implementation 'org.springframework.boot:spring-boot-starter-validation'`

**Jakarta Bean Validation**

`jakarta.validation-api` : Bean Validation 인터페이스
`hibernate-validator` 구현체

**검증 애노테이션**

- `@NotBlank` : 빈값 + 공백만 있는 경우를 허용하지 않는다.
- `@NotNull` : null 을 허용하지 않는다.
- `@Range(min = 1000, max = 1000000)` : 범위 안의 값이어야 한다.
- `@Max(9999)` : 최대 9999까지만 허용한다.

**스프링 MVC는 어떻게 Bean Validator를 사용?**

스프링 부트가 `spring-boot-starter-validation` 라이브러리를 넣으면 자동으로 Bean Validator를 인지하고 스프링에 통합한다

**스프링 부트는 자동으로 글로벌 Validator로 등록한다.**

`LocalValidatorFactoryBean` 을 글로벌 Validator로 등록한다. 이 Validator는 `@NotNull` 같은
애노테이션을 보고 검증을 수행한다. 이렇게 글로벌 Validator가 적용되어 있기 때문에, `@Valid` ,
`@Validated` 만 적용하면 된다.
검증 오류가 발생하면, `FieldError` , `ObjectError` 를 생성해서 `BindingResult`에 담아준다.

**검증 순서**

1. `@ModelAttribute` 각각의 필드에 타입 변환 시도
    1. 성공하면 다음으로
    2. 실패하면 `typeMismatch` 로 `FieldError` 추가
2. Validator 적용

## Bean Validation - 에러 코드

`NotBlank` 라는 오류 코드를 기반으로 `MessageCodesResolver` 를 통해 다양한 메시지 코드가 순서대로 생성된다.

**@NotBlank**

- NotBlank.item.itemName
- NotBlank.itemName
- NotBlank.java.lang.String
- NotBlank

**@Range**

- Range.item.price
- Range.price
- Range.java.lang.Integer
- Range

**BeanValidation 메시지 찾는 순서**

1. 생성된 메시지 코드 순서대로 messageSource 에서 메시지 찾기
2. 애노테이션의 message 속성 사용 @NotBlank(message = "공백! {0}")
3. 라이브러리가 제공하는 기본 값 사용 공백일 수 없습니다.

## Bean Validation - 한계

데이터를 등록할 때와 수정할 때는 요구사항이 다를 수 있다.

**등록시 기존 요구사항**

- 수량 : 최대9999

**수정시 요구사항**

- 수량 : 무제한

## **Bean Validation - groups**

**방법 2가지**

- BeanValidation의 groups 기능을 사용한다.
- Item을 직접 사용하지 않고, ItemSaveForm, ItemUpdateForm 같은 폼 전송을 위한 별도의 모델 객체를 만들어서 사용한다.

**저장용 groups 생성**

```java
public interface SaveCheck {
}
```

수정영 **groups 생성**

```java
public interface UpdateCheck{
}
```

Item - groups 적용

```java
import lombok.Data;
import org.hibernate.validator.constraints.Range;
import javax.validation.constraints.Max;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
@Data
public class Item {
	 @NotNull(groups = UpdateCheck.class) //수정시에만 적용
	 private Long id;

	 @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})
	 private String itemName;

	 @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
	 @Range(min = 1000, max = 1000000, groups = {SaveCheck.class,
	UpdateCheck.class})
	 private Integer price;

	 @NotNull(groups = {SaveCheck.class, UpdateCheck.class})
	 @Max(value = 9999, groups = SaveCheck.class) //등록시에만 적용
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

## Form 전송 객체 분리

**폼 데이터 전달에 Item 도메인 객체 사용**

- `HTML Form -> Item -> Controller -> Item -> Repository`
    - 장점: Item 도메인 객체를 컨트롤러, 리포지토리 까지 직접 전달해서 중간에 Item을 만드는 과정이 없어서 간단하다.
    - 단점: 간단한 경우에만 적용할 수 있다. 수정시 검증이 중복될 수 있고, groups를 사용해야 한다.

**폼 데이터 전달을 위한 별도의 객체 사용**

- `HTML Form -> ItemSaveForm -> Controller -> Item 생성 -> Repository`
    - 장점: 전송하는 폼 데이터가 복잡해도 거기에 맞춘 별도의 폼 객체를 사용해서 데이터를 전달 받을 수 있다. 보통 등록과, 수정용으로 별도의 폼 객체를 만들기 때문에 검증이 중복되지 않는다.
    - 단점: 폼 데이터를 기반으로 컨트롤러에서 Item 객체를 생성하는 변환 과정이 추가된다.

## HTTP 메시지 컨버터

`@Valid` , `@Validated` 는 `HttpMessageConverter` `( @RequestBody )`에도 적용할 수 있다

```java
@Slf4j
@RestController
@RequestMapping("/validation/api/items")
public class ValidationItemApiController {

	 @PostMapping("/add")
	 public Object addItem(@RequestBody @Validated ItemSaveForm form,
	 BindingResult bindingResult) {

			log.info("API 컨트롤러 호출");

			if (bindingResult.hasErrors()) {
				 log.info("검증 오류 발생 errors={}", bindingResult);
				 return bindingResult.getAllErrors();
			}
			
			log.info("성공 로직 실행");
			return form;
 }
}
```

**API의 경우 3가지 경우를 나누어 생각해야 한다.**

- 성공 요청: 성공
- 실패 요청: JSON을 객체로 생성하는 것 자체가 실패함
- 검증 오류 요청: JSON을 객체로 생성하는 것은 성공했고, 검증에서 실패함

**@ModelAttribute vs @RequestBody**

HTTP 요청 파리미터를 처리하는 `@ModelAttribute` 는 각각의 필드 단위로 세밀하게 적용된다. 그래서 특정 필드에 타입이 맞지 않는 오류가 발생해도 나머지 필드는 정상 처리할 수 있었다.
`HttpMessageConverter` 는 `@ModelAttribute` 와 다르게 각각의 필드 단위로 적용되는 것이 아니라, 전체 객체 단위로 적용된다.
따라서 메시지 컨버터의 작동이 성공해서 `ItemSaveForm` 객체를 만들어야 `@Valid` , `@Validated` 가적용된다.
