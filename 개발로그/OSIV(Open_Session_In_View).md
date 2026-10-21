## 0. OSIV

OSIV = Open Session In View

JPA 기준으로 풀어 말하면 HTTP 요청이 끝날 때까지 영속성 컨텍스트(EntityManager)를 열어두는 전략

## 1. 증상

권한 취약점 이슈 대응으로 `SessionInterceptor.preHandle()` 안에서 `X-Set-Id` 헤더의 세트 접근 권한을
검사하도록 바꾼 뒤, **세트를 전환하는 첫 요청**에서 500 에러가 발생.

```
org.hibernate.LazyInitializationException:
  failed to lazily initialize a collection of role:
  com.jiransnc.vada.entity.domain.InspectionSetEntity.users
  - could not initialize proxy - no Session
```

터지는 지점은 `PermissionChecker.hasUserSet()` 의 아래 라인들.

```java
List<UserEntity> users = inspectionSet.getUsers();          // LAZY 컬렉션
List<GroupUserEntity> inspectionGroupUsers = inspectionSet.getGroupUsers();
List<GroupUserEntity> userGroupUsers = user.getGroupUsers();
```

특징적으로 **같은 메서드를 컨트롤러에서 호출하는 경로(`SetController`, `UserController`,
`ApiValidService`)는 아무 문제가 없었고, 인터셉터에서 호출하는 경로에서만** 재현되었다.

---

## 2. 원인

### 2-1. 조회한 엔티티는 이미 준영속(detached) 상태였다

`UserFindServiceImpl`, `InspectionSetFindServiceImpl` 에는 `@Transactional` 이 없다.

```java
@Service
public class InspectionSetFindServiceImpl implements InspectionSetFindService {
    public InspectionSetEntity getInspectionSet(int setId) {
        return inspectionSetRepository.findById(setId) ... // 여기서만 트랜잭션이 열렸다 닫힘
    }
}
```

`SimpleJpaRepository` 의 메서드에는 `@Transactional(readOnly = true)` 가 붙어 있으므로,
호출하는 쪽에 트랜잭션이 없으면 **`findById()` 한 번마다 트랜잭션/EntityManager 가 열렸다가 즉시 닫힌다.**
반환된 엔티티는 그 순간 준영속이 되고, 이후 `getUsers()` 같은 LAZY 컬렉션을 건드리면
프록시를 초기화할 Session 이 없어 `LazyInitializationException` 이 난다.

> `InspectionSetEntity.users`, `InspectionSetEntity.groupUsers`, `UserEntity.groupUsers` 는
> 모두 `@ManyToMany` 이고, **`@ManyToMany` 의 기본 fetch 전략은 LAZY** 다.

### 2-2. 그런데도 지금까지 동작했던 이유 = OSIV

`application.properties` 에 `spring.jpa.open-in-view` 설정이 없다 → **기본값 true(OSIV 켜짐)**.
OSIV 가 켜져 있으면 요청 시작 시점에 EntityManager 를 만들어 스레드에 바인딩하고 요청이 끝날 때까지 유지한다.
그래서 서비스가 트랜잭션 없이 조회해도 엔티티가 요청 내내 영속 상태로 남아 **컨트롤러/뷰 계층에서 LAZY 초기화가 그냥 됐던 것**이다.

즉, 지금까지 잘 돌아간 것은 코드가 옳아서가 아니라 **OSIV가 뒤를 봐주고 있었기 때문**이다.

### 2-3. 인터셉터는 OSIV가 아직 시작되지 않은 구간이다 ← 진짜 원인

Spring Boot 2.x 는 OSIV 를 **필터가 아니라 `OpenEntityManagerInViewInterceptor`(HandlerInterceptor)** 로 등록한다.
`spring-boot-autoconfigure-2.7.18.jar` 의 `JpaBaseConfiguration$JpaWebConfiguration` 를 확인해 보면:

```java
public OpenEntityManagerInViewInterceptor openEntityManagerInViewInterceptor();
public WebMvcConfigurer openEntityManagerInViewInterceptorConfigurer(OpenEntityManagerInViewInterceptor interceptor);
// 익명 WebMvcConfigurer( JpaWebConfiguration$1 ) → registry.addWebRequestInterceptor(interceptor)
```

이 익명 `WebMvcConfigurer` 는 `Ordered` 를 구현하지도, `@Order` 를 달고 있지도 않다.
우리 `HttpInterceptorConfig` 역시 순서 지정이 없다. 결과적으로 두 Configurer 는 같은 우선순위로 취급되고,
**사용자 정의 @Configuration 빈이 자동설정 빈보다 먼저 수집**되므로 인터셉터 체인은 다음 순서가 된다.

```
[요청]
  └ DispatcherServlet
      ├ 1. SessionInterceptor.preHandle()          ← EntityManager 바인딩 없음 (OSIV 시작 전)
      │     └ permissionChecker.hasUserSet()  ✗ LazyInitializationException
      ├ 2. OpenEntityManagerInViewInterceptor.preHandle()  ← 여기서 EM 이 바인딩됨
      └ 3. Controller / Service                   ← 그래서 여기서는 LAZY 가 잘 됐다
```

`SetController`, `UserController`, `ApiValidService` 에서의 호출은 3번 구간이라 문제가 없었고,
같은 `hasUserSet()` 을 1번 구간으로 옮긴 순간 바로 터진 것이다.

**정리하면 "코드가 갑자기 잘못된 것"이 아니라, 항상 잘못돼 있던 코드를 OSIV 보호막 밖으로 꺼낸 것이다.**

---

## 3. 조치

`PermissionChecker.hasUserSet()` 에 트랜잭션 경계를 직접 부여했다. (커밋 `d55f5792d`)

```java
+import org.springframework.transaction.annotation.Transactional;
...
+    @Transactional(readOnly = true)
     public void hasUserSet(int userId, int setId) throws Exception {
```

- 메서드 진입 시 트랜잭션과 EntityManager 가 생성되어 **메서드가 끝날 때까지 영속성 컨텍스트가 유지**된다.
- `getUser()` / `getInspectionSet()` 두 조회가 **하나의 영속성 컨텍스트를 공유**하므로 반환 엔티티가 영속 상태다.
- 따라서 `getUsers()`, `getGroupUsers()` LAZY 초기화가 메서드 안에서 정상 수행된다.
- OSIV 유무와 무관하게 동작한다. → 추후 `spring.jpa.open-in-view=false` 로 바꿔도 안전.
- `PermissionChecker` 는 인터페이스가 없는 `@Component` → CGLIB 프록시가 생성되고, 호출자가 모두 외부 빈이므로
  자기호출(self-invocation)로 AOP가 무력화되는 문제는 없다.
- `readOnly = true` 이므로 `VADAException`(RuntimeException) 이 던져져도 롤백 부담이 없다.

---

## 4. 검증

1. 로그인 후 세트 A 진입 → 세트 B 로 전환 (`X-Set-Id` 헤더가 바뀌는 첫 요청)
2. 500 / `LazyInitializationException` 없이 정상 응답, 세션 `SET_ID` 갱신 확인
3. 권한 없는 세트 ID 를 `X-Set-Id` 로 넣어 호출 → `SET_NOT_PERMISSION` 이 정상 반환되는지 확인
   (VIM-1489 원래 목적인 권한 상승 차단이 유지되는지)
4. 기존 경로(`SetController`, `ApiValidService`) 회귀 확인

---

## 5. 대안 및 향후 개선 과제

| 방안 | 내용 | 평가 |
|---|---|---|
| **A. `@Transactional` 부여** (채택) | 권한 체크 메서드에 트랜잭션 경계 부여 | 변경 최소, 즉시 해결. 다만 컬렉션 전체를 로딩 |
| B. `exists` 카운트 쿼리로 전환 | `select count(*)` 로 "권한 있음" 여부만 확인 | 성능상 최선. 세트 사용자/그룹이 많아지면 A는 매번 전체 컬렉션 로딩 → **후속 개선 권장** |
| C. `@EntityGraph` / fetch join 조회 | 조회 시점에 필요한 연관을 함께 가져옴 | N+1 도 함께 해소되나 조회 메서드가 용도별로 분화됨 |
| D. OSIV 인터셉터 순서 조정 / `OpenEntityManagerInViewFilter` 로 교체 | 인터셉터 구간까지 EM 을 열어둠 | 전역 영향이 커서 이 이슈만으로 건드리기엔 과함 |

추가로 검토할 것:

- `spring.jpa.open-in-view` 가 명시돼 있지 않아 기동 시 Spring Boot 의 WARN 로그가 남는다.
  정책을 정해 **명시적으로 설정**하는 것이 좋다. (끄려면 위 A/B/C 같은 트랜잭션 경계 정리가 선행돼야 함)
- 엔티티 조회 서비스(`*FindServiceImpl`)들이 트랜잭션 없이 리포지토리만 호출하는 구조라
  동일 문제가 다른 지점에서도 재발할 수 있다.

---

## 6. 교훈 — OSIV 밖에서는 LAZY를 건드리지 말 것

아래 구간들은 **OSIV의 영속성 컨텍스트가 없거나 이미 닫힌 구간**이다.
이 안에서 엔티티를 다루려면 반드시 자체 트랜잭션 경계(`@Transactional`)를 두거나, DTO/fetch join 으로 미리 가져와야 한다.

- `HandlerInterceptor.preHandle()` (OSIV 인터셉터보다 먼저 실행)
- `Filter` (Boot 2.x 는 OSIV 가 인터셉터라 필터 구간은 밖)
- `@Async` / `@Scheduled` / 별도 스레드 (요청 스레드가 아니므로 바인딩된 EM 없음)
- `afterCompletion()` 이후, 응답 커밋 이후 로직
