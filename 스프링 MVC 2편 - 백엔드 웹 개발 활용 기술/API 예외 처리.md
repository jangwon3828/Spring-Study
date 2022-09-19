# API 예외 처리

## API 예외 처리 - 시작

HTML 페이지의 경우 지금까지 설명했던 것 처럼 4xx, 5xx와 같은 오류 페이지만 있으면 대부분의 문제를해결할 수 있다. 

그런데 API의 경우에는 생각할 내용이 더 많다. 오류 페이지는 단순히 고객에게 오류 화면을 보여주고 끝이지만, API는 각 오류 상황에 맞는 오류 응답 스펙을 정하고, JSON으로 데이터를 내려주어야 한다

```java
@RequestMapping(value = "/error-page/500", produces =MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<Map<String, Object>> errorPage500Api(HttpServletRequest request, HttpServletResponse response) {
    log.info("API errorPage 500");
    Map<String, Object> result = new HashMap<>();
    Exception ex = (Exception) request.getAttribute(ERROR_EXCEPTION);
    result.put("status", request.getAttribute(ERROR_STATUS_CODE));
    result.put("message", ex.getMessage());
    Integer statusCode = (Integer)
    request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
    return new ResponseEntity(result, HttpStatus.valueOf(statusCode));
}
```

```java
{
 "message": "잘못된 사용자",
 "status": 500
}
```

## API 예외 처리 - 스프링 부트 기본 오류 처리

```java
@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse 
response) {}
@RequestMapping
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {}
```

`/error` 동일한 경로를 처리하는 `errorHtml()` , `error()` 두 메서드를 확인할 수 있다.

- `errorHtml()` : `produces = MediaType.TEXT_HTML_VALUE` : 클라이언트 요청의 Accept 해더 값이 `text/html` 인 경우에는 `errorHtml()` 을 호출해서 view를 제공한다.
- `error()` : 그외 경우에 호출되고 `ResponseEntity` 로 HTTP Body에 JSON 데이터를 반환한다

```json
{
 "timestamp": "2021-04-28T00:00:00.000+00:00",
 "status": 500,
 "error": "Internal Server Error",
 "exception": "java.lang.RuntimeException",
 "trace": "java.lang.RuntimeException: 잘못된 사용자\n\tat 
hello.exception.web.api.ApiExceptionController.getMember(ApiExceptionController
.java:19...,
 "message": "잘못된 사용자",
 "path": "/api/members/ex"
}
```

## API 예외 처리 - HandlerExceptionResolver 시작

**HandlerExceptionResolver**

스프링 MVC는 컨트롤러(핸들러) 밖으로 예외가 던져진 경우 예외를 해결하고, 동작을 새로 정의할 수 있는 방법을 제공한다. 컨트롤러 밖으로 던져진 예외를 해결하고, 동작 방식을 변경하고 싶으면 `HandlerExceptionResolver` 를 사용하면 된다. 줄여서 `ExceptionResolver` 라 한다

```java
@Slf4j
public class MyHandlerExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            if (ex instanceof IllegalStateException) {
                log.info("IllegalStateException resolver to 400");
                response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage());
                return new ModelAndView();
            }
        } catch (IOException e) {
            log.error("resolver ex", e);
        }

        return null;
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        resolvers.add(new MyHandlerExceptionResolver());
    }
}
```

- `ExceptionResolver` 가 `ModelAndView` 를 반환하는 이유는 마치 `try, catch`를 하듯이, `Exception` 을 처리해서 정상 흐름 처럼 변경하는 것이 목적이다. 이름 그대로 `Exception` 을 `Resolver(해결)`하는 것이 목적이다.
- 여기서는 `IllegalArgumentException` 이 발생하면 `response.sendError(400)` 를 호출해서 `HTTP`상태 코드를 400으로 지정하고, 빈 `ModelAndView` 를 반환한다.
- 반환 값에 따른 동작 방식
    - `HandlerExceptionResolver` 의 반환 값에 따른 `DispatcherServlet` 의 동작 방식은 다음과 같다.
    - 빈 `ModelAndView: new ModelAndView()` 처럼 빈 `ModelAndView` 를 반환하면 뷰를 렌더링 하지않고, 정상 흐름으로 서블릿이 리턴된다.
- `ModelAndView` 지정: `ModelAndView` 에 `View , Model` 등의 정보를 지정해서 반환하면 뷰를 렌더링한다.
- `null`: `null` 을 반환하면, 다음 `ExceptionResolver` 를 찾아서 실행한다. 만약 처리할 수 있는 `ExceptionResolver` 가 없으면 예외 처리가 안되고, 기존에 발생한 예외를 서블릿 밖으로 던진다.
- **ExceptionResolver 활용**
    - **예외 상태 코드 변환**
        - 예외를 `response.sendError(xxx)` 호출로 변경해서 서블릿에서 상태 코드에 따른 오류를 처리하도록 위임
        - 이후 `WAS`는 서블릿 오류 페이지를 찾아서 내부 호출, 예를 들어서 스프링 부트가 기본으로 설정한 `/error` 가 호출됨
    - **뷰 템플릿 처리**
        - `ModelAndView` 에 값을 채워서 예외에 따른 새로운 오류 화면 뷰 렌더링 해서 고객에게 제공
    - **API 응답 처리**
        - `response.getWriter().println("hello");` 처럼 `HTTP` 응답 바디에 직접 데이터를 넣어주는 것도 가능하다. 여기에 `JSON` 으로 응답하면 `API` 응답 처리를 할 수 있다

## API 예외 처리 - HandlerExceptionResolver 활용

- 예외가 발생하면 `WAS`까지 예외가 던져지고, `WAS`에서 오류 페이지 정보를 찾아서 다시 `/error`를 호출하는 과정은 생각해보면 너무 복잡하다.
- `ExceptionResolver` 를 활용하면 예외가 발생했을 때 이런 복잡한 과정 없이 여기에서 문제를 깔끔하게 해결할 수 있다.

```java
public class UserException extends RuntimeException{

    public UserException() {
        super();
    }

    public UserException(String message) {
        super(message);
    }

    public UserException(String message, Throwable cause) {
        super(message, cause);
    }

    public UserException(Throwable cause) {
        super(cause);
    }

    protected UserException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
}

@Slf4j
public class UserHandlerExceptionResolver implements HandlerExceptionResolver {

    private final ObjectMapper objectMapper = new ObjectMapper();
    private static final String APPLICATION_JSON = "application/json";
    public static final String UTF_8 = "utf-8";
    public static final String ERROR_500 = "error/500";

    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try{
            if (ex instanceof UserException) {
                log.info("UserException resolver to 400");
                String acceptHeader = request.getHeader("accept");
                response.setStatus(HttpServletResponse.SC_BAD_REQUEST);

                if (APPLICATION_JSON.equals(acceptHeader)) {
                    Map<String, Object> errorResult = new HashMap<>();
                    errorResult.put("ex", ex.getClass());
                    errorResult.put("message", ex.getMessage());

                    String result = objectMapper.writeValueAsString(errorResult);

                    response.setContentType(APPLICATION_JSON);
                    response.setCharacterEncoding(UTF_8);
                    response.getWriter().write(result);

                    return  new ModelAndView();
                } else {
                    // TEXT/HTML
                    return new ModelAndView(ERROR_500);
                }
            }
        }catch (IOException e){
            log.error("resolver ex",e);
        }
        return null;
    }
}
```

- `ExceptionResolver` 를 사용하면 컨트롤러에서 예외가 발생해도 `ExceptionResolver` 에서 예외를 처리해버린다.
- 따라서 예외가 발생해도 서블릿 컨테이너까지 예외가 전달되지 않고, 스프링 `MVC`에서 예외 처리는 끝이난다.
- 결과적으로 `WAS` 입장에서는 정상 처리가 된 것이다. 이렇게 예외를 이곳에서 모두 처리할 수 있다는 것이 핵심이다

## API 예외 처리 - 스프링이 제공하는 ExceptionResolver1

**스프링 부트가 기본으로 제공하는 `ExceptionResolver` 는 다음과 같다.**
`HandlerExceptionResolverComposite` 에 다음 순서로 등록

1. `ExceptionHandlerExceptionResolver`
    - `@ExceptionHandler` 을 처리한다. API 예외 처리는 대부분 이 기능으로 해결한다.
2. `ResponseStatusExceptionResolver`
    - HTTP 상태 코드를 지정해준다.
    - 예) `@ResponseStatus(value = HttpStatus.NOT_FOUND)`
    - @ResponseStatus 가 달려있는 예외
        - 예외에 다음과 같이 @ResponseStatus 애노테이션을 적용하면 HTTP 상태 코드를 변경해준다.
        
        ```java
        @ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")
        public class BadRequestException extends RuntimeException {
        }
        ```
        
    - ResponseStatusException 예
        - `@ResponseStatus` 는 개발자가 직접 변경할 수 없는 예외에는 적용할 수 없다. (애노테이션을 직접 넣어야 하는데, 내가 코드를 수정할 수 없는 라이브러리의 예외 코드 같은 곳에는 적용할 수 없다.) 추가로 애노테이션을 사용하기 때문에 조건에 따라 동적으로 변경하는 것도 어렵다. 이때는 `ResponseStatusException` 예외를 사용하면 된다
3. `DefaultHandlerExceptionResolver` → 우선 순위가 가장 낮다
    - 스프링 내부 기본 예외를 처리한다.
    - 대표적으로 파라미터 바인딩 시점에 타입이 맞지 않으면 내부에서`TypeMismatchException`이 발생하는데, 이 경우 예외가 발생했기 때문에 그냥 두면 서블릿 컨테이너까지 오류가 올라가고, 결과적으로 500 오류가 발생한다. 하지만 HTTP 요청 정보를 잘못 호출해서 발생하는 문제라 500 오류가 아니라 HTTP 상태 코드 400 오류로 변경한다
    
    ## API 예외 처리 - @ExceptionHandler
    
    **@ExceptionHandler**
    
    스프링은 API 예외 처리 문제를 해결하기 위해 `@ExceptionHandler` 라는 애노테이션을 사용하는 매우 편리한 예외 처리 기능을 제공하는데, 이것이 바로 `ExceptionHandlerExceptionResolver` 이다.
    스프링은 `ExceptionHandlerExceptionResolver` 를 기본으로 제공하고, 기본으로 제공하는
    `ExceptionResolver` 중에 우선순위도 가장 높다. 실무에서 API 예외 처리는 대부분 이 기능을 사용한다
    
    ```java
    @Slf4j
    @RestController
    public class ApiExceptionV2Controller {
    	 @ResponseStatus(HttpStatus.BAD_REQUEST)
    	 @ExceptionHandler(IllegalArgumentException.class)
    	 public ErrorResult illegalExHandle(IllegalArgumentException e) {
    		 log.error("[exceptionHandle] ex", e);
    		 return new ErrorResult("BAD", e.getMessage());
    	 }
    
    	 @ExceptionHandler
    	 public ResponseEntity<ErrorResult> userExHandle(UserException e) {
    		 log.error("[exceptionHandle] ex", e);
    		 ErrorResult errorResult = new ErrorResult("USER-EX", e.getMessage());
    		 return new ResponseEntity<>(errorResult, HttpStatus.BAD_REQUEST);
    	 }
    
    	 @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    	 @ExceptionHandler
    	 public ErrorResult exHandle(Exception e) {
    		 log.error("[exceptionHandle] ex", e);
    		 return new ErrorResult("EX", "내부 오류");
    	 }
    	 @GetMapping("/api2/members/{id}")
    	 public MemberDto getMember(@PathVariable("id") String id) {
    		 if (id.equals("ex")) {
    			 throw new RuntimeException("잘못된 사용자");
    		 }
    
    		 if (id.equals("bad")) {
    			 throw new IllegalArgumentException("잘못된 입력 값");
    		 }
    		 if (id.equals("user-ex")) {
    			 throw new UserException("사용자 오류");
    		 }
    		return new MemberDto(id, "hello " + id);
     }
    
     @Data
     @AllArgsConstructor
     static class MemberDto {
    		 private String memberId;
    		 private String name;
    	}
    }
    ```
    
    ## **API 예외처리 - @ControllerAdvice**
    
    - `@ControllerAdvice` 는 대상으로 지정한 여러 컨트롤러에 `@ExceptionHandler` , `@InitBinder` 기능을 부여해주는 역할을 한다.
    - `@ControllerAdvice` 에 대상을 지정하지 않으면 모든 컨트롤러에 적용된다. (글로벌 적용)
    - `@RestControllerAdvice` 는 `@ControllerAdvice` 와 같고, `@ResponseBody` 가 추가되어 있다.
    - `@Controller` , `@RestController` 의 차이와 같다.
    
    ```java
    @Slf4j
    @RestControllerAdvice
    public class ExControllerAdvice {
    
        @ResponseStatus(HttpStatus.BAD_REQUEST)
        @ExceptionHandler(IllegalStateException.class)
        public ErrorResult illegalExHandler(IllegalStateException e) {
            log.error("[exceptionHandler] ex", e);
            return new ErrorResult("BAD", e.getMessage());
        }
    
        @ExceptionHandler
        public ResponseEntity<ErrorResult> userHandler(UserException e) {
            log.error("[exceptionHandler] ex", e);
            ErrorResult errorResult = new ErrorResult("USER-EX", e.getMessage());
            return new ResponseEntity(errorResult, HttpStatus.BAD_REQUEST);
        }
    
        @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
        @ExceptionHandler
        public ErrorResult exHandler(Exception e) {
            log.error("[exceptionHandler] ex", e);
            return new ErrorResult("EX", "내부 오류");
        }
    }
    ```
    
    - **대상 컨트롤러 지정 방법**
    
    ```java
    // Target all Controllers annotated with @RestController
    @ControllerAdvice(annotations = RestController.class)
    public class ExampleAdvice1 {}
    
    // Target all Controllers within specific packages
    @ControllerAdvice("org.example.controllers")
    public class ExampleAdvice2 {}
    
    // Target all Controllers assignable to specific classes
    @ControllerAdvice(assignableTypes = {ControllerInterface.class,
    AbstractController.class})
    public class ExampleAdvice3 {}
    ```
    
