# 스프링 MVC - 구조 이해

## 스프링 MVC - 구조

- **DispatcheServlet 구조**
    - `org.springframework.web.servlet.DispatcherServlet`
    - 스프링 MVC 컨트롤러 == 디스패처 서블릿
- **DispacherServlet 서블릿 등록**
    - `DispacherServlet` 도 부모 클래스에서 `HttpServlet` 을 상속 받아서 사용하고, 서블릿으로 동작한다.
        - `DispatcherServlet` → `FrameworkServlet` → `HttpServletBean` → `HttpServlet`
    - 스프링 부트는 `DispacherServlet` 을 서블릿으로 자동으로 등록하면서 모든 경로 `( urlPatterns="/" )`에대해서 매핑한다.
        - 참고: 더 자세한 경로가 우선순위가 높다. 그래서 기존에 등록한 서블릿도 함께 동작한다.
- **요청 흐름**
    - 서블릿이 호출되면 `HttpServlet` 이 제공하는 `serivce()` 가 호출된다.
    - 스프링 MVC는 `DispatcherServlet` 의 부모인 `FrameworkServlet` 에서 `service()` 를 오버라이드해두었다.`FrameworkServlet.service()` 를 시작으로 여러 메서드가 호출되면서 `DispacherServlet.doDispatch()` 가 호출된다.
- **동작 순서**
    1. **핸들러 조회:** 핸들러 매핑을 통해 요청 URL에 매핑된 핸들러(컨트롤러)를 조회한다.
    2. **핸들러 어댑터 조회:** 핸들러를 실행할 수 있는 핸들러 어댑터를 조회한다.
    3. **핸들러 어댑터 실행:** 핸들러 어댑터를 실행한다.
    4. **핸들러 실행:** 핸들러 어댑터가 실제 핸들러를 실행한다.
    5. **ModelAndView 반환:** 핸들러 어댑터는 핸들러가 반환하는 정보를 ModelAndView로 변환해서반환한다.
    6. **viewResolver 호출:** 뷰 리졸버를 찾고 실행한다
        1. JSP의 경우: InternalResourceViewResolver 가 자동 등록되고, 사용된다.
    7. **View 반환:** 뷰 리졸버는 뷰의 논리 이름을 물리 이름으로 바꾸고, 렌더링 역할을 담당하는 뷰 객체를반환한다.
        1. JSP의 경우 InternalResourceView(JstlView) 를 반환하는데, 내부에 forward() 로직이 있다.
    8. **뷰 렌더링:** 뷰를 통해서 뷰를 렌더링 한다.
- 주요 인터페이스 목록
    - 핸들러 매핑: org.springframework.web.servlet.HandlerMapping
    - 핸들러 어댑터: org.springframework.web.servlet.HandlerAdapter
    - 뷰 리졸버: org.springframework.web.servlet.ViewResolver
    - 뷰: org.springframework.web.servlet.View

## 핸들러 매핑과 핸들러 어댑터

- **Controller 인터페이스**
    - **HandlerMapping(핸들러 매핑)**
        - 핸들러 매핑에서 이 컨트롤러를 찾을 수 있어야 한다.
        - 예) **스프링 빈의 이름으로 핸들러를 찾을 수 있는 핸들러 매핑**이 필요하다

```java
0 = RequestMappingHandlerMapping : 애노테이션 기반의 컨트롤러인 @RequestMapping에서 사용
1 = BeanNameUrlHandlerMapping : 스프링 빈의 이름으로 핸들러를 찾는다
```

- **HandlerAdapter(핸들러 어댑터)**
    - 핸들러 매핑을 통해서 찾은 핸들러를 실행할 수 있는 핸들러 어댑터가 필요하다.
    - 예) `Controller` 인터페이스를 실행할 수 있는 핸들러 어댑터를 찾고 실행해야 한다.

```java
0 = RequestMappingHandlerAdapter : 애노테이션 기반의 컨트롤러인 @RequestMapping에서 사용
1 = HttpRequestHandlerAdapter : HttpRequestHandler 처리
2 = SimpleControllerHandlerAdapter : Controller 인터페이스(애노테이션X, 과거에 사용) 처리
```

1. **핸들러 매핑으로 핸들러 조회**
    1. `HandlerMapping` 을 순서대로 실행해서, 핸들러를 찾는다.
    2. 이 경우 빈 이름으로 핸들러를 찾아야 하기 때문에 이름 그대로 빈 이름으로 핸들러를 찾아주는 `BeanNameUrlHandlerMapping` 가 실행에 성공하고 핸들러인 `OldController` 를 반환한다.
2. **핸들러 어댑터 조회**
    1. `HandlerAdapter` 의 `supports()` 를 순서대로 호출한다.
    2. `SimpleControllerHandlerAdapter` 가 `Controller` 인터페이스를 지원하므로 대상이 된다.
3. **핸들러 어댑터 실행**
    1. 디스패처 서블릿이 조회한 `SimpleControllerHandlerAdapter` 를 실행하면서 핸들러 정보도 함께 넘겨준다.
    2. `SimpleControllerHandlerAdapter` 는 핸들러인 `OldController` 를 내부에서 실행하고, 그 결과를반환한다

## **뷰 리졸버**

- **뷰 리졸버 - `InternalResourceViewResolver`**
    - 스프링 부트는 `InternalResourceViewResolver` 라는 뷰 리졸버를 자동으로 등록하는데, 이때`application.properties` 에 등록한 `spring.mvc.view.prefix` , `spring.mvc.view.suffix` 설정 정보를 사용해서 등록한다
- **스프링 부트가 자동 등록하는 뷰 리졸버**

```java
1 = BeanNameViewResolver : 빈 이름으로 뷰를 찾아서 반환한다. (예: 엑셀 파일 생성기능에 사용)
2 = InternalResourceViewResolver : JSP를 처리할 수 있는 뷰를 반환한다.
```

1. **핸들러 어댑터 호출**
    1. 핸들러 어댑터를 통해 `new-form` 이라는 논리 뷰 이름을 획득한다.
2. **ViewResolver 호출**
    1. `new-form` 이라는 뷰 이름으로 `viewResolver`를 순서대로 호출한다.
    2. `BeanNameViewResolver` 는 `new-form` 이라는 이름의 스프링 빈으로 등록된 뷰를 찾아야 하는데 없다.
    3. `InternalResourceViewResolver` 가 호출된다.
3. `InternalResourceViewResolver`
    1. 뷰 리졸버는 `InternalResourceView` 를 반환한다.
4. **뷰 - `InternalResourceView`**
    1. `InternalResourceView` 는 JSP처럼 포워드 `forward()` 를 호출해서 처리할 수 있는 경우에 사용한다.
5. **view.render()**
    1. `view.render()` 가 호출되고 `InternalResourceView` 는 `forward()` 를 사용해서 JSP를 실행한다.
