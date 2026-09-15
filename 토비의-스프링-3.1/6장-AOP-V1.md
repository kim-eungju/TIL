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
