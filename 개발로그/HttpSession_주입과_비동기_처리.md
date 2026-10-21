## 1. HttpSession은 주입받을 수 있다

```java
@Component
@RequiredArgsConstructor
public class FileTicketStoreImpl {
    private final HttpSession session; // 별도 설정 없이 주입됨
}
```
- Spring MVC가 기본적으로 제공하는 기능이다
- 같은 방식으로 주입 가능한 타입은 `HttpServletRequest`, `HttpServletResponse`, `HttpSession`, `WebRequest` 등이 있다

---

## 2. 동작 원리: 진짜 세션이 아니라 프록시

요청 도착
- `DispatcherServlet`이 현재 요청을 `RequestContextHolder(ThreadLocal)`에 저장
- 싱글톤 빈 안의 session 필드 = 프록시
- `session.getId()` 호출 순간, 프록시가 `RequestContextHolder`에서
  - "현재 스레드의 요청"을 꺼내 그 요청의 세션에 위임
- 요청 종료 시 스프링이 `RequestContextHolder` 정리
- 빈은 하나지만 호출하는 스레드의 요청에 따라 다른 세션으로 동작함
- 바탕은 `ThreadLocal`이다. 그래서 요청 스레드 안에서만 유효함

--- 

## 3. 인자로 넘기기 vs 주입

| 항목 | 주입 (프록시) | 인자로 넘기기 |
|---|---|---|
| 코드 | 시그니처가 깔끔함 | 컨트롤러에서 서비스까지 계속 전달 |
| 의존성 | 숨겨져 있음 | 명시적으로 드러남 |
| @Async, 스케줄러 | ❌ 예외 발생 | ✅ 동작함 |
| 세션이 없을 때 | 새 세션을 만들어 버림 | 호출하는 쪽이 결정 |
| 단위 테스트 | mock 요청 컨텍스트 필요 | 값만 넣으면 됨 |
| 적합한 위치 | 웹 요청 전용 인프라 구현체 | 서비스, 도메인, 비동기 로직 |

---

## 4. 비동기(@Async)에서 주입된 세션을 쓰면?

기본 설정에서는 응답을 먼저 보냈는지와 상관없이 비동기 메서드에서 호출하는 순간 예외가 발생함
- `@Async` 메서드는 별도 스레드 풀의 스레드에서 실행된다
- `RequestContextHolder`는 일반 `ThreadLocal`을 사용하므로 요청 정보가 다른 스레드로 전달되지 않는다
- 따라서 주입받은 `session.getId()` 호출 시 `IllegalStateException: No thread-bound request found`가 발생
- HTTP 응답 여부와 관계없이 비동기 스레드에는 처음부터 요청 정보가 없기 때문에 동일한 예외가 발생합니다.

> 비동기 메서드에서는 인자로 넘기자. threadContextInheritable=true 옵션을 켜면 예외는 사라지지만, 응답이 끝난 뒤 호출하면 요청 객체가 재사용되거나, 최악의 경우 다른 사용자의 요청과 세션을 읽을 수도 있다
