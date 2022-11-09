# @Aspect
- AspectJ에서 제공하는 애노테이션으로, 어드바이저를 편리하게 생성하는 기능을 제공한다.
- 자동 프록시 생성기가 `@Aspect`가 붙은 스프링 빈을 찾아서 `Advisor`로 만들어준다. (자동 프록시 생성기는 스프링 빈으로 등록된 `Advisor`도 자동으로 찾아서 프록시를 만들어준다.)
- 과정
  1. 스프링 애플리케이션 로딩시에 자동 프록시 생성기를 호출한다.
  2. 자동 프록시 생성기는 `@Aspect`가 붙은 스프링 빈을 찾는다.
  3. `@Aspect` 어드바이저 빌더라는 것을 통해 `@Aspect` 애노테이션 정보 기반으로 `Advisor`를 생성한다.
  4. 생성한 `Advisor`를 @Aspect` 어드바이저 빌더 내부에 저장한다.


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
