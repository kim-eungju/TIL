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
