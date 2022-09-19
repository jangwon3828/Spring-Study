# 검증1 - Validation

## 검증 요구사항

**요구사항: 검증 로직 추가**

- 타입 검증
    - 가격, 수량에 문자가 들어가면 검증 오류 처리
- 필드 검증
    - 상품명: 필수, 공백X
    - 가격: 1000원 이상, 1백만원 이하
    - 수량: 최대 9999
- 특정 필드의 범위를 넘어서는 검증
    - 가격 * 수량의 합은 10,000원 이상

**컨트롤러의 중요한 역할중 하나는 HTTP 요청이 정상인지 검증하는 것이다.** 그리고 정상 로직보다 이런 검증 로직을 잘 개발하는 것이 어쩌면 더 어려울 수 있다

**참고: 클라이언트 검증, 서버 검증**

- 클라이언트 검증은 조작할 수 있으므로 보안에 취약하다.
- 서버만으로 검증하면, 즉각적인 고객 사용성이 부족해진다.
- 둘을 적절히 섞어서 사용하되, 최종적으로 서버 검증은 필수
- API 방식을 사용하면 API 스펙을 잘 정의해서 검증 오류를 API 응답 결과에 잘 남겨주어야 함

## **BindingResult**

- 스프링이 제공하는 검증 오류를 보관하는 객체이다. 검증 오류가 발생하면 여기에 보관하면 된다.
- BindingResult 가 있으면 @ModelAttribute 에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출된다!

**예) @ModelAttribute에 바인딩 시 타입 오류가 발생하면?**

- BindingResult 가 없으면 400 오류가 발생하면서 컨트롤러가 호출되지 않고, 오류 페이지로 이동한다.
- BindingResult 가 있으면 오류 정보( FieldError )를 BindingResult 에 담아서 컨트롤러를 정상 호출한다

**BindingResult에 검증 오류를 적용하는 3가지 방법**

- `@ModelAttribute` 의 객체에 타입 오류 등으로 바인딩이 실패하는 경우 스프링이 `FieldError` 생성해서 `BindingResult` 에 넣어준다.
- 개발자가 직접 넣어준다.
- `Validator` 사용 이것은 뒤에서 설명

**BindingResult와 Errors**

`org.springframework.validation.Errors`
`org.springframework.validation.BindingResult`

`BindingResult` 는 인터페이스이고, `Errors` 인터페이스를 상속받고 있다.
실제 넘어오는 구현체는 `BeanPropertyBindingResult` 라는 것인데, 둘다 구현하고 있으므로 `BindingResult` 대신에 `Errors`를 사용해도 된다. `Errors` 인터페이스는 단순한 오류 저장과 조회 기능을 제공한다. `BindingResult` 는 여기에 더해서 추가적인 기능들을 제공한다. `addError()` 도`BindingResult` 가 제공하므로 여기서는 `BindingResult` 를 사용하자. 주로 관례상 `BindingResult` 를
많이 사용한다.

## FieldError, ObjectError

**FieldError 생성자**

`FieldError` 는 두 가지 생성자를 제공한다.

```java
public FieldError(String objectName, String field, String defaultMessage);
public FieldError(String objectName, String field, @Nullable Object 
rejectedValue, boolean bindingFailure, @Nullable String[] codes, @Nullable
Object[] arguments, @Nullable String defaultMessage)
```

**파라미터 목록**

- `objectName` : 오류가 발생한 객체 이름
- `field` : 오류 필드
- `rejectedValue` : 사용자가 입력한 값(거절된 값)
- `bindingFailure` : 타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
- `codes` : 메시지 코드
- `arguments` : 메시지에서 사용하는 인자
- `defaultMessage` : 기본 오류 메시지

### **`rejectValue()` , `reject()`**

`BindingResult` 가 제공하는 `rejectValue()` , `reject()` 를 사용하면 `FieldError` , `ObjectError` 를 직접 생성하지 않고, 깔끔하게 검증 오류를 다룰 수 있다

### 오류 코드 level

오류 코드를 만들 때 다음과 같이 자세히 만들 수도 있고,
`required.item.itemName` : 상품 이름은 필수 입니다.
`range.item.price` : 상품의 가격 범위 오류 입니다.
또는 다음과 같이 단순하게 만들 수도 있다.
`required` : 필수 값 입니다.
`range` : 범위 오류 입니다.

그런데 오류 메시지에 `required.item.itemName` 와 같이 객체명과 필드명을 조합한 세밀한 메시지 코드가 있으면 이 메시지를 높은 우선순위로 사용하는 것이다

```java
#Level1
required.item.itemName: 상품 이름은 필수 입니다.
#Level2
required: 필수 값 입니다.
```

### MessageCodesResolver

스프링은 `MessageCodesResolver` 라는 것으로 이러한 기능을 지원한다.

- 검증 오류 코드로 메시지 코드들을 생성한다.
- `MessageCodesResolver` 인터페이스이고 `DefaultMessageCodesResolver` 는 기본 구현체이다.
- 주로 다음과 함께 사용 `ObjectError` , `FieldError`

**FieldError** `rejectValue("itemName", "required")`

다음 4가지 오류 코드를 자동으로 생성

- `required.item.itemName`
- `required.itemName`
- `required.java.lang.String`
- `required`

**ObjectError** `reject("totalPriceMin")`

다음 2가지 오류 코드를 자동으로 생성

- `totalPriceMin.item`
- `totalPriceMin`

### 오류 코드 관리 전략

**핵심은 구체적인 것에서! 덜 구체적인 것으로!**
`MessageCodesResolver` 는 `required.item.itemName` 처럼 구체적인 것을 먼저 만들어주고, `required` 처럼 덜 구체적인 것을 가장 나중에 만든다.
이렇게 하면 앞서 말한 것 처럼 메시지와 관련된 공통 전략을 편리하게 도입할 수 있다.

### 스프링이 직접 만든 오류 메시지 처리

검증 오류 코드는 다음과 같이 2가지로 나눌 수 있다.

- 개발자가 직접 설정한 오류 코드 → `rejectValue()` 를 직접 호출
- 스프링이 직접 검증 오류에 추가한 경우(주로 타입 정보가 맞지 않음)

**ValidationUtils**

- `Empty`. 공백 같은 단순한 기능에 사용된다.

`ValidationUtils.rejectIfEmptyOrWhitespace(bindingResult, "itemName","required");`
