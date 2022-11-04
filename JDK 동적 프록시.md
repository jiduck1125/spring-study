# JDK 동적 프록시

- 프록시 기능을 만들기 위해 데코레이터 패턴 등을 이용하면 적용할 클래스 수만큼 프록시 클래스를 만들어야 한다.
- 동적 프록시를 사용하면 프록시 클래스를 직접 만들지 않고 런타임에 동적으로 만들어 준다.
- JDK 동적 프록시는 인터페이스 기반으로 동작하기 때문에 인터페이스가 필수이다.
- 자바 언어에서 기본 제공한다.

# JDK 동적 프록시 사용

**InvocationHandler**

- `InvocationHandler` 인터페이스를 구현하고 `invoke` 메서드를 오버라이딩해서 공통 로직을 작성하면 된다.
- 리플렉션을 이용해 `method.invoke(target, args)`로 `target`의 메서드를 호출한다.

```java
@Slf4j
public class TimeInvocationHandler implements InvocationHandler {

    private final Object target;

    public TimeInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log.info("TimeProxy 실행");
        long startTime = System.currentTimeMillis();

        Object result = method.invoke(target, args);    // target 메서드 실행

        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("TimeProxy 종료 resultTime={}", resultTime);

        return result;
    }
}
```

**인터페이스**

```java
public interface AInterface {
    String call();
}
```

**구현**

```java
@Slf4j
public class AImpl implements AInterface {
    @Override
    public String call() {
        log.info("A 호출");
        return "a";
    }
}
```

**실행**

- `Proxy.newProxyInstance` 메서드에 클래스 로더 정보, 인터페이스, 핸들러 로직을 넣어주면 자동으로 해당 인터페이스 기반인 동적 프록시를 생성하고 결과를 반환해준다.
- 클라이언트가 프록시를 통해 `call()`을 실행하면, 프록시가 `InvocationHandler.invoke()`를 호출한다. 여기서는 구현체인 `TimeInvocationHandler.invoke()`가 호출된다.
- `TimeInvocationHandler.invoke()`에서 로직을 실행하고 내부에서 `method.invoke(target, args)`를 호출하면 `target.call()`, 즉 `AImpl.call()`이 호출된다.

```java
AInterface target = new AImpl();

TimeInvocationHandler handler = new TimeInvocationHandler(target);

AInterface proxy = (AInterface) Proxy.newProxyInstance(AInterface.class.getClassLoader(), new Class[]{AInterface.class}, handler);

proxy.call();
```

## 장점

- JDK 동적 프록시를 사용하면 프록시 적용 대상 클래스 개수만큼 프록시 클래스를 만들지 않아도 된다. 프록시 클래스를 동적으로 만들어 준다.
- 동일한 부가 로직은 한번만 개발해서 공통으로 적용할 수 있다.

## 단점

- 인터페이스를 필수로 만들어야 한다.
