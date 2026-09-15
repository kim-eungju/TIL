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
