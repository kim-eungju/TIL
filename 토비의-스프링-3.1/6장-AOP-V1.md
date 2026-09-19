## 1. 트랜잭션 코드가 너무 거슬린다

유저 승급 서비스가 있다고 가정해보자

```java
public void upgradeLevels() {
    TransactionStatus status =
        transactionManager.getTransaction(...);
    try {
        // --- 핵심 비즈니스 로직 시작 ---
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
        // --- 핵심 비즈니스 로직 종료 ---
        transactionManager.commit(status);
    } catch (RuntimeException e) {
        transactionManager.rollback(status);
        throw e;
    }
}
```

아래와 같은 기술적인 코드가 들어와 있다

```
트랜잭션 시작
commit
rollback
```

__핵심 관심사(Business Logic)__ 와 __부가 관심사(Transcation)__ 가 하나의 메서드안에 섞여 있는 상태다

---

## 2. 일단 메서드로 분리해보자

코드는 좀 깨끗해졌지만, 근본적인 문제는 해결이 안됐다

```java
public void upgradeLevels() {
    // transaction begin
    try {
        upgradeLevelsInternal();
        // commit
    } catch (Exception e) {
        // rollback
    }
}

private void upgradeLevelsInternal() {
    // 순수 비즈니스 로직
}
```
`UserService` 클래스가 여전히 두 가지 책임을 가지고 있다
```
UserService
 ├─ 사용자 레벨 업그레이드
 └─ 트랜잭션 처리
```

---

## 3. DI를 이용해서 트랜잭션을 클래스 밖으로 빼자

여기서 인터페이스를 이용한다
```java
public interface UserService {

    void upgradeLevels();
}
```

실제 비즈니스 로직
```java
public class UserServiceImpl implements UserService {

    @Override
    public void upgradeLevels() {
        // 순수 비즈니스 로직
    }
}
```

그리고 트랜잭션을 담당하는 클래스 `UserServiceTx`를 따로 만든다
```java
public class UserServiceTx implements UserService {

    private UserService userService;
    private PlatformTransactionManager transactionManager;

    @Override
    public void upgradeLevels() {
        TransactionStatus status =
            transactionManager.getTransaction(...);
        try {
            // --- 핵심 비즈니스 로직 시작 ---
            userService.upgradeLevels();
            // --- 핵심 비즈니스 로직 종료 ---
            transactionManager.commit(status);
        } catch (RuntimeException e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

호출 구조는 이렇게 된다

```
Controller
    ↓
UserServiceTx
    ↓
UserServiceImpl
    ↓
UserDao
```

여기서 `UserServiceTx`가 __프록시(Proxy)__ 역할을 한다
이 구조가 정말 중요하다

---

## 4. Proxy

프록시란? 진짜 객체 대신 클라이언트 요청을 먼저 받아주는 객체
```
Client
   ↓
Proxy
   ↓
Target
```

여기서는 이렇게 된다
```
Proxy  = UserServiceTx
Target = UserServiceImpl
```

Proxy는 요청을 받아서 트랜잭션을 처리하고 실제 비즈니스 로직은 Target에게 위임한다

```
proxy.upgradeLevels() 의 내부
---
transaction begin
target.upgradeLevels()
transaction commit
---
```

토비에서 이후 계속 나오는 Target은 __부가기능을 적용할 실제 객체__ 라고 생각하면 된다

---

## 5. 그런데 프록시 클래스를 매번 만들면?

서비스가 100개라면? <br/>
이런 클래스를 전부 만들어야 한다
```
UserServiceTx
OrderServiceTx
PaymentServiceTx
AssetServiceTx
...
```

이러면 똑같은 코드를 계속 작성해야 한다
```java
try {
    target.doSomething();
    commit();
} catch (...) {
    rollback();
}
```

굉장히 비효율적인데 __프록시를 자동으로 만들 수 없을까?__ <br/>
여기서 __JDK Dynamic Proxy__ 가 등장한다

---

## 6. Dynamic Proxy

Java에는 런타임에 프록시 객체를 만들어주는 기능이 있다 (`InvocationHandler`) <br/>

우선, 아래 코드는 `createdUser()`, `deleteUser()`마다 똑같은 프록시 코드를 반복한다
```java
public class UserServiceProxy implements UserService {

    private final UserService target;

    public UserServiceProxy(UserService target) {
        this.target = target;
    }

    @Override
    public void createUser(String name) {
        System.out.println("트랜잭션 시작");

        try {
            target.createUser(name);
            System.out.println("트랜잭션 커밋");
        } catch (Exception e) {
            System.out.println("트랜잭션 롤백");
            throw e;
        }
    }

    @Override
    public void deleteUser(Long id) {
        System.out.println("트랜잭션 시작");

        try {
            target.deleteUser(id);
            System.out.println("트랜잭션 커밋");
        } catch (Exception e) {
            System.out.println("트랜잭션 롤백");
            throw e;
        }
    }
}
```

이걸 JDK Dynamic Proxy 로 바꾸면 `UserServiceProxy` 클래스 자체를 없애고, `InvocationHandler`를 만든다

```java
public class TransactionInvocationHandler implements InvocationHandler {

    private final Object target;

    public TransactionInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(
            Object proxy,
            Method method,
            Object[] args
    ) throws Throwable {

        System.out.println("트랜잭션 시작");

        try {
            Object result = method.invoke(target, args);

            System.out.println("트랜잭션 커밋");

            return result;

        } catch (Exception e) {
            System.out.println("트랜잭션 롤백");
            throw e;
        }
    }
}
```
```java
public static void main(String[] args) {

    // 진짜 객체
    UserService target = new UserServiceImpl();

    // Dynamic Proxy 생성
    UserService proxy = (UserService) Proxy.newProxyInstance(
            UserService.class.getClassLoader(),
            new Class[]{UserService.class},
            new TransactionInvocationHandler(target)
    );

    proxy.createUser("kim");

    proxy.deleteUser(1L);
}
```

이제 프록시 클래스를 직접 만들지 않아도된다
```
프록시 객체 생성 = Java에게 맡김
부가기능 = InvocationHandler에 작성
```

---

## 7. 그래도 문제가 하나 남아 있다

Spring Bean으로 관리할 수 없다
```java
UserService proxy = (UserService) Proxy.newProxyInstance(
            UserService.class.getClassLoader(),
            new Class[]{UserService.class},
            new TransactionInvocationHandler(target)
    );
```

그래서 Spring은 프록시를 편하게 생성할 수 있도록 `ProxyFactoryBean` 추상화를 제공한다 <br/>
여기서 개념을 한 단계 더 분리한다
```
Target
  ↓
Advice
  ↓
Pointcut
```

Advice 와 Pointcut 이 두 단어가 굉장히 중요하다

---

## 8. Advice & Pointcut 

Advice
- 무엇을 적용할 것인가?
- 트랜잭션 처리
- 부가 기능

Pointcut
- 어디에 적용할 것인가?
- 트랜잭션 Advice를 어떤 메서드에 적용할건데?

Advisor
- Pointcut + Advice 둘을 합쳐 __Advisor__ 라고 부른다

---

## 9. 자동 프록시 생성

그런데 `ProxyFactoryBean`도 `Bean` 마다 등록해야 한다 <br/>
서비스가 100개라면 `Bean` 100개를 설정해야 한다
```
UserService
OrderService
PaymentService
AssetService
...
```
```java
@Bean
public ProxyFactoryBean userService(
        UserService userServiceTarget,
        LoggingAdvice loggingAdvice) {

    ProxyFactoryBean factory = new ProxyFactoryBean();

    factory.setTarget(userServiceTarget);
    factory.addAdvice(loggingAdvice);

    return factory;
}
...
```

자동 프록시 생성은 Spring 컨테이너가 `Bean`을 만들 때 아래와 같이 동작한다

```
Bean 생성
   ↓
이 Bean이 Pointcut 대상인가?
   ↓ YES
Proxy 생성
   ↓
Proxy를 Bean으로 등록
```

pointcut 대상 판단은 내가 직접 지정한다 <br/>
수동으로 `Bean` 을 만들때와 다르게, __"규칙"__ 을 지정할 수 있다
```java
@Bean
public Advisor loggingAdvisor(LoggingAdvice advice) {

    // 1. Pointcut 생성
    NameMatchMethodPointcut pointcut =
            new NameMatchMethodPointcut();

    // save 메서드가 대상
    pointcut.setMappedName("save");

    // 2. Pointcut + Advice를 Advisor로 등록
    return new DefaultPointcutAdvisor(
            pointcut,
            advice
    );
}
```

---

## 10. AOP

여기까지 오면 구조가 이렇게 된다
```
              ┌──────────────┐
              │ Transaction  │
              │    Advice    │
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │   Pointcut   │
              └──────┬───────┘
                     │
                     ▼

UserService     OrderService     AssetService
    │                │                │
    ▼                ▼                ▼
 business          business          business
```

AOP는 이것을 __Transaction Aspect__ 라는 하나의 모듈로 모은다 <br/>
그래서 __Aspect Oriented Programming__ 관점지향 프로그래밍이라고 부른다 _*Aspect = 측면_ <br/><br/> 
AOP는 OOP를 대체하는게 아니라, <br/>
OOP만으로 깔끔하게 모듈화하기 어려운 이런 횡단 관심사를 보완하는 기술이라고 이해하는게 정확하다

---

## 11. @Transactional

현대 Spring 개발자는 익숙하다 

```java
@Transactional
public void upgradeLevels() {
    ...
}
```

코드 한 줄이지만 실제 개념적인 구조는 대략 이렇게 된다

```java
Caller
   ↓
Spring Proxy
   ↓
Transaction Interceptor
   ↓
TransactionManager
   ↓
Target.upgradeLevels()
```

즉 `@Transactional`을 보고 Spring이 메서드 안에 코드를 집어넣는게 아님
```
트랜잭션시작
   ↓
methodB();
   ↓
트랜잭션종료
```

그래서 유명한 문제가 생긴다 <br/>
`methodA()`에 트랜잭션이 없는 상태에서, 내부 호출로 실행된 `methodB()`의 `@Transactional`은 동작하지 않는다
```java
@Service
public class UserService {

    public void methodA() {
        methodB(); // this.methodB()
    }

    @Transactional
    public void methodB() {
        ...
    }
}
```

비유하면
- 프록시 = 건물 입구의 보안요원
- @Transactional = "들어갈 때 보호 장비를 지급해 주세요"라는 표시
- 외부에서 methodB() 호출 = 건물 입구를 통과함 → 보호 장비 지급
- methodA()에서 methodB() 호출 = 이미 건물 안에서 방을 이동함 → 입구를 다시 안 지나감

트랜잭션 메서드는 별도 빈으로 분리하자
```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserTransactionService transactionService;

    public void methodA() {
        transactionService.methodB();
    }
}

@Service
public class UserTransactionService {

    @Transactional
    public void methodB() {
        ...
    }
}
```

> `@Transactional`은 메서드 자체의 마법이 아니라, 스프링 프록시가 메서드 호출 앞뒤에 트랜잭션 처리를 끼워 넣는 기능이다. 같은 객체 내부 호출은 프록시를 거치지 않는다.

---

## 12. 테스트

단위테스트도 AOP와 연결돼 있다 <br/>
기존 `UserService`가 다른 로직과 강하게 얽혀있으면 테스트하기 어렵다

```
UserService
 ├ Business Logic
 ├ Transaction
 ├ UserDao
 └ MailSender
```

관심사를 분리하면 순수 비즈니스 로직만 고립해서 테스트할 수 있다
```
UserServiceImpl
   ↓
MockUserDao
MockMailSender
```

> 관심사를 분리하면 설계도 좋아지고 테스트도 쉬워진다.

---

## 13. 결론

관심사 분리 → DI → Proxy → Dynamic Proxy → ProxyFactory → 자동 Proxy → AOP <br/>
이 흐름을 이해하면 `@Transactional`, Spring Security의 메서드 보안, `@Cacheable`, 커스텀 `@Aspect` 등을 볼 때 <br/>
"아, 결국 타깃 앞에 프록시 세워서 호출을 가로채는 구조구나" 생각하자
