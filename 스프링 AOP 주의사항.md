# 프록시 내부 호출

## 문제
- 스프링 AOP는 프록시 방식으로 동작한다. 프록시를 통해서 target을 호출한다.
- 프록시를 거치지 않고 target을 직접 호출하게 되면 AOP가 적용되지 않고, 어드바이스도 적용되지 않는다.
- 스프링 빈에 등록된 프록시 객체를 호출하면 문제가 없지만, 대상 객체 내부에서 자신을 호출하게 되면 프록시를 거치지 않기 때문에 AOP가 적용되지 않는 문제가 발생한다.

## 대안

### 자기 자신 주입
- 자기 자신을 의존관계 주입 받아서 호출하는 방식이다. 주입받는건 프록시 객체이므로 AOP가 적용된다.
- 생성자 주입으로 받으면 순환 참조 오류가 발생하므로 setter로 주입받아야 한다. 스프링 부트 2.6부터는 setter로 받는것도 순환 참조 오류가 발생하는데, `properties`에 `spring.main.allow-circular-references=true` 옵션을 추가해줘야 오류가 안난다.

```java
@Slf4j
@Component
public class CallService {

    private CallService callService;

    @Autowired
    public void setCallService(CallService callService) {
        this.callService = callService;
    }

    public void external() {
        callService.internal(); // 프록시 객체를 통해 호출
    }

    public void internal() {
        log.info("call internal");
    }
}
```

### 지연 조회
- 실제 객체를 사용하는 시점에 스프링 컨테이너에서 객체를 가져오는 방식이다.
- `ObjectProvider` 또는 `ApplicationContext`를 사용해서 가져올 수 있다.

```java
@Slf4j
@Component
public class CallService {

    private final ObjectProvider<CallService> callServiceProvider;

    public CallService(ObjectProvider<CallService> callServiceProvider) {
        this.callServiceProvider = callServiceProvider;
    }

    public void external() {
        CallService callService = callServiceProvider.getObject();
        callService.internal();
    }

    public void internal() {
        log.info("call internal");
    }
}
```

### 구조 변경
- 내부 호출이 발생하지 않도록 구조를 변경하는 것이다. 가장 권장하는 방법이다.
- 내부에서 호출하지 않도록 클래스를 분리하고 스프링 빈을 주입받아서 사용하면 된다.
- 아니면 클라이언트에서 호출을 각각 하도록 변경할 수도 있다.

```java
@Slf4j
@Component
public class InternalService {

    public void internal() {
        log.info("call internal");
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class CallService {

    private final InternalService internalService;

    public void external() {
        internalService.internal();
    }
}
```

# 프록시 기술 한계

## 타입 캐스팅
- JDK 동적 프록시는 인터페이스를 기반으로 프록시를 생성한다.
- CGLIB는 구체 클래스를 상속받아서 프록시를 생성한다.
- JDK 동적 프록시를 사용해 프록시를 생성하면 구체 클래스로 타입 캐스팅이 불가능하다. 프록시는 인터페이스를 기반으로 만들어졌기 때문에 구체 클래스는 알지 못하기 때문이다.

## 의존관계 주입
- 구체 클래스로 의존관계 주입을 받으려고 할 경우, JDK 동적 프록시를 사용하면 구체 클래스로의 타입 캐스팅이 되지 않기 때문에 오류가 발생한다.

## CGLIB의 문제점
CGLIB는 구체 클래스를 상속받기 때문에 상속에 따른 문제점이 존재한다.

- 대상 클래스에 기본 생성자 필수
  - 자바에서는 상속을 받으면 자식 클래스의 생성자를 호출할 때 부모 클래스의 생성자도 호출해야 한다.(`super()`)
  - CGLIB 프록시는 대상 클래스를 상속 받고, 생성자에서 대상 클래스의 기본 생성자를 호출한다. 따라서 대상 클래스에 기본 생성자가 필수이다.
- 생성자 2번 호출 문제
  - CGLIB를 사용하면 실제 target의 객체를 생성할 때 1번, 프록시 객체를 생성할 때 부모 클래스의 생성자를 호출하므로 1번, 총 2번 생성자를 호출하게 된다.
- final 키워드 클래스, 메서드 사용 불가
  - 클래스에 final 키워드가 있으면 상속이 불가능하고, 메서드에 final이 있으면 오버라이딩이 불가능하다. CGLIB는 상속을 사용하므로 프록시가 생성되지 않거나 정상 동작하지 않는다.

### 스프링의 해결책
- 스프링 3.2부터 CGLIB를 스프링 내부에 함께 패키징하여 라이브러리 추가 없이 사용할 수 있게 되었다.
- 스프링 4.0부터 `objenesis`라는 라이브러리를 사용해서 기본 생성자 없이 객체 생성이 가능하게 되었고, 생성자도 1번만 호출된다.
- 스프링 부트 2.0부터 CGLIB를 기본으로 사용하도록 해서 구체 클래스로 의존관계 주입이 안되는 문제를 해결했다.
  - JDK 동적 프록시를 사용하려면 `application.properties`에 다음과 같이 설정해주면 된다.
  ```
  spring.aop.proxy-target-class=false
  ```
