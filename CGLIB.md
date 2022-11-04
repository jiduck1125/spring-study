# CGLIB (Code Generator Library)

- 바이트 코드를 조작해서 동적으로 클래스를 생성할 수 있다.
- 인터페이스가 없이 구체 클래스를 상속해서 프록시를 동적으로 만들어낸다.
- 원래는 외부 라이브러리인데 스프링 내부 소스에도 포함되어 있다.
- CGLIB는 직접 사용할 일은 거의 없다. `ProxyFactory`를 사용하는 것이 편리하다.

# 사용

- `MethodInterceptor` 인터페이스를 구현하고 `intercept` 메서드를 오버라이딩해서 공통 로직을 작성하면 된다.
- `methodProxy.invoke(target, args)`로 `target`의 메서드를 호출한다.

**InvocationHandler**

```java
@Slf4j
public class TimeMethodInterceptor implements MethodInterceptor {

    private final Object target;

    public TimeMethodInterceptor(Object target) {
        this.target = target;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        log.info("TimeProxy 실행");
        long startTime = System.currentTimeMillis();

        Object result = methodProxy.invoke(target, args);

        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("TimeProxy 종료 resultTime={}", resultTime);

        return result;
    }
}
```

**구체 클래스**

```java
@Slf4j
public class ConcreteService {
    public void call() {
        log.info("ConcreteService 호출");
    }
}
```

**실행**

- `Enhancer`를 사용해 프록시를 생성한다.
- `setSuperclass`로 상속할 구체 클래스를 지정하고, `setCallback`으로 프록시에 적용할 실행 로직을 할당한다.
- `create`를 실행하면 구체 클래스를 상속받은 프록시가 생성된다.

```java
ConcreteService target = new ConcreteService();

Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(ConcreteService.class);
enhancer.setCallback(new TimeMethodInterceptor(target));
ConcreteService proxy = (ConcreteService) enhancer.create();

proxy.call();
```

## 단점

GCLIB는 상속을 사용하므로 제약이 있다.

- GCLIB가 자식 클래스를 동적으로 생성하기 때문에 기본 생성자가 필요하다.
- 클래스에 final이 붙으면 상속이 불가능하므로 예외가 발생한다.
- 메서드에 final이 붙으면 오버라이딩이 불가능하므로 프록시 로직이 동작하지 않는다.
