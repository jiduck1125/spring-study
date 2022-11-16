# @Aspect
- AspectJ에서 제공하는 애노테이션으로, 어드바이저를 편리하게 생성하는 기능을 제공한다.
- 자동 프록시 생성기가 `@Aspect`가 붙은 스프링 빈을 찾아서 `Advisor`로 만들어준다. (자동 프록시 생성기는 스프링 빈으로 등록된 `Advisor`도 자동으로 찾아서 프록시를 만들어준다.)
- 과정
  1. 스프링 애플리케이션 로딩시에 자동 프록시 생성기를 호출한다.
  2. 자동 프록시 생성기는 `@Aspect`가 붙은 스프링 빈을 찾는다.
  3. `@Aspect` 어드바이저 빌더라는 것을 통해 `@Aspect` 애노테이션 정보 기반으로 `Advisor`를 생성한다.
  4. 생성한 `Advisor`를 `@Aspect` 어드바이저 빌더 내부에 저장한다.


# @Aspect 사용

**Aspect**
- `@Aspect` 애노테이션을 붙이고, `@Around`에 AspectJ 포인트컷 표현식을 넣으면 된다. 메서드 내용이 `Advice` 로직이 된다.
- `joinPoint.proceed()`로 target을 호출한다.

```java
@Aspect
public class LogTraceAspect {

    private final LogTrace logTrace;

    public LogTraceAspect(LogTrace logTrace) {
        this.logTrace = logTrace;
    }

    @Around("execution(* hello.proxy.app..*(..))")
    public Object execute(ProceedingJoinPoint joinPoint) throws Throwable {
        TraceStatus status = null;
        try {
            String message = joinPoint.getSignature().toShortString();
            status = logTrace.begin(message);

            // target 호출
            Object result = joinPoint.proceed();

            logTrace.end(status);
            return result;
        } catch (Exception e) {
            logTrace.exception(status, e);
            throw e;
        }
    }
}
```

**Config**
- `@Aspect`를 스프링 빈으로 등록한다.
```java
@Configuration
public class AopConfig {

    @Bean
    public LogTraceAspect logTraceAspect(LogTrace logTrace) {
        return new LogTraceAspect(logTrace);
    }
}
```

## 포인트컷 분리
- `@Pointcut`으로 포인트컷을 따로 분리해서 재사용할 수 있다.
- 포인트컷을 애스펙트 클래스 내부에 같이 넣어도 되고, 별도 클래스로 분리해도 된다.

**Pointcut**

- `@Ponitcut`에 포인트컷 표현식을 넣으면 된다.
- 메서드의 반환 타입은 `void`여야 하고, 메서드 내용은 비워두면 된다.
- 메서드 이름과 파리미터를 합쳐서 포인트컷 시그니처라고 한다. (ex. `allOrder()`)
```java
public class Pointcuts {

    @Pointcut("execution(* hello.aop.order..*(..))")
    public void allOrder() {}
}
```

**Aspect**

- 포인트컷을 지정할 때 포인트컷 시그니처를 사용하면 된다. 외부 클래스에 있는 포인트컷은 패키지명까지 붙여줘야 한다.
```java
@Slf4j
@Aspect
public class AspectV4Pointcut {

    @Around("hello.aop.order.aop.Pointcuts.allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }
}
```

## 포인트컷 조합
- 포인트컷은 `&&`(AND), `||`(OR),  `!`(NOT) 이 3가지로 조합해서 사용할 수 있다.
```java
@Around("allOrder() && allService()")
```

## 어드바이스 순서
- 어드바이스는 기본적으로 순서를 보장하지 않는다.
- `@Order`를 사용해서 순서를 지정할 수 있다. 하지만 클래스 단위로만 적용이 가능하기 때문에 애스펙트를 별도 클래스로 분리해야 한다.

다음과 같이 설정하면 `TxAspect`, `LogAspect` 순서로 실행된다.
```java
@Aspect
@Order(2)
public static class LogAspect {
    @Around("hello.aop.order.aop.Pointcuts.allOrder()")
    public Object doLog(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("[log] {}", joinPoint.getSignature());
        return joinPoint.proceed();
    }
}

@Aspect
@Order(1)
public static class TxAspect {
    @Around("hello.aop.order.aop.Pointcuts.orderAndService()")
    public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {

        try {
            log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
            Object result = joinPoint.proceed();
            log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
            return result;
        } catch (Exception e) {
            log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
            throw e;
        } finally {
            log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
        }
    }
}
```

## 어드바이스 종류

### @Around
- 메서드 호출 전후에 수행된다.
- 조인포인트 실행 여부를 선택하거나, 전달값, 반환값을 변환하거나, 예외를 변환할 수 있다.
- 다른 어드바이스는 `@Around`가 실행할 수 있는것의 일부분만 제공하는 것이다.
- `@Around`는 첫번째 파라미터로 `ProceedingJoinPoint`를 사용해야 한다.
- 그 외 어드바이스는 `JoinPoint`를 첫번째 파라미터에 사용할 수 있다. (생략도 가능)
- `proceed()`로 타겟을 호출할 수 있다. 여러번 호출할 수도 있다.
```java
@Around("hello.aop.order.aop.Pointcuts.orderAndService()")
public Object doTransaction(ProceedingJoinPoint joinPoint) throws Throwable {

    try {
        // @Before
        log.info("[트랜잭션 시작] {}", joinPoint.getSignature());
        Object result = joinPoint.proceed();
        // @AfterReturning
        log.info("[트랜잭션 커밋] {}", joinPoint.getSignature());
        return result;
    } catch (Exception e) {
        // @AfterThrowing
        log.info("[트랜잭션 롤백] {}", joinPoint.getSignature());
        throw e;
    } finally {
        // @After
        log.info("[리소스 릴리즈] {}", joinPoint.getSignature());
    }
}
```

### @Before
- 조인포인트 실행 전에 실행되고, 메서드가 종료되면 자동으로 다음 타겟이 호출된다.
```java
@Before("hello.aop.order.aop.Pointcuts.orderAndService()")
public void doBefore(JoinPoint joinPoint) {
    log.info("[before] {}", joinPoint.getSignature());
}
```
### @AfterReturning
- 메서드 실행이 정상적으로 반환될 때 실행된다.
- `returning` 속성에 넣은 이름이 어드바이스 메서드의 매개변수 이름과 일치해야 한다.
- 리턴되는 객체를 변경할 수는 없지만 조작은 가능하다.
```java
@AfterReturning(value = "hello.aop.order.aop.Pointcuts.orderAndService()", returning = "result")
public void doReturn(JoinPoint joinPoint, Object result) {
    log.info("[return] {} return={}", joinPoint.getSignature(), result);
}
```
### @AfterThrowing
- 메서드 실행이 예외를 던져서 종료될 때 실행된다.
- `returning` 속성에 넣은 이름이 어드바이스 메서드의 매개변수 이름과 일치해야 한다.
```java
@AfterThrowing(value = "hello.aop.order.aop.Pointcuts.orderAndService()", throwing = "ex")
public void doThrowing(JoinPoint joinPoint, Exception ex) {
    log.info("[ex] {} message={}", ex);
}
```
### @After
- 메서드 실행이 종료되면 실행된다. 메서드 실행이 정상적이든 예외를 던지든 상관없이 실행된다.
- 일반적으로 리소스를 해제하는데 사용된다.
```java
@After(value = "hello.aop.order.aop.Pointcuts.orderAndService()")
public void doAfter(JoinPoint joinPoint) {
    log.info("[after] {}", joinPoint.getSignature());
}
```

### 순서
- 스프링 5.2.7 버전부터 동일한 `@Aspect` 안에서 동일한 조인포인트의 적용 우선순위는 다음과 같다.
- `@Around`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`

# 포인트컷 지시자

## execution

### execution 형식
- `?`는 생략 가능한 부분이고, `*`로 패턴을 지정할 수도 있다.
- execution(접근제어자? 반환타입 선언타입?메서드이름(파라미터) 예외?)
```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern)
          throws-pattern?)
```



## within

## args

## @target, @within

## @annotation, @args

## bean

## 매개변수 전달

## this, target