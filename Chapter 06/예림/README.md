# 6장. 동시성, 데이터가 꼬이기 전에 잡아야 한다
## 서버와 동시 실행
- 서버는 동시에 클라이언트의 요청을 처리하며 DB도 동시에 여러 쿼리를 실행한다.
  <img width="559" height="233" alt="스크린샷 2026-03-08 오후 5 23 26" src="https://github.com/user-attachments/assets/197ac81a-51d1-4075-9ddc-1a6b7465c0b4" />
- 동시에 처리해야 처리량과 응답 시간을 줄일 수 있음
- 서버가 동시에 여러 클라이언트의 요청을 처리하는 방식 2가지
  - 클라이언트 요청마다 스레드를 할당해서 처리
    - 여러 스레드가 동시에 코드 실행
  - 비동기 IO (또는 논블로킹 IO)를 사용해서 처리
    - 이때도 여러 스레드 사용 (특히 IO)
- race condition
    - 동시성을 고려하지 않으면 데이터에 오류가 발생할 수 있다.
- 동시성 문제는 미묘해서 재현이 안되는 경우가 많으므로 염두에 두고 개발하는 것이 필요

## 잘못된 데이터 공유로 인한 문제 예시
```java
public class PayService {
  ...
  public PayResp pay(PayRequest req) {
    ...
    this.payId = getPayId(); // 단계 1
    saveTemp(this.payId, req); // 단계 2
    PayResp resp = sendPayData(this.payId, ...); // 단계 3
    applyResponse(resp); // 단계 4
    return resp;
  }

  public void applyResponse(PayResp resp) {
    PayData payData = createPayDataFromResp(resp); // 단계 4-1
    updatePayData(this.payId, payData); // 단계 4-2
}
```
- PayService가 싱글톤 객체라면 다중 스레드 환경에서 단계 1 payId 값과 단계 4-2 payId 값이 다를 수 있다.
  - 다음 스레드가 payId 값을 덮어쓸 수 있음
- DB 또한 동시성을 고려하지 않으면 데이터 일관성에 문제가 생길 수 있다.
- 관리자와 고객이 동시에 주문 정보를 변경할 때
  - 관리자는 주문 상태를 `배송`으로 변경하고 고객은 같은 주문을 `춰소`
  - DB 관점에서는 주문을 취소 상태로 변경 후 배송 상태로 변경하는 상황이 생길 수 있음

## 프로세스 수준에서의 동시 접근 제어
### lock을 이용한 접근 제어
1. 잠금 획득
2. 공유 자원에 접근 (임계 영역)
3. 잠금 해제

- ReentrantLock을 사용해 동시에 HashMap을 수정하는 것을 막는 코드
  ```java
  public class UserSessions {
    private Lock lock = new ReentrantLock();
    private Map<String, UserSession> sessions = new HashMap<>(); // ExecutorService를 생성
    // ExecutorService는 동시에 500개 작업을 실행할 수 있다.

    public void addUserSession(UserSession session) {
      lock.lock(); // 잠금 획득할 때까지 대기
      try {
        sessions.put(session.getSessionId(), session); // 공유 자원 접근
      } finally {
        lock.unlock(); // 잠금 해제
      }
    }

    public UserSession getUserSession(String sessionId) {
      lock.lock();
      try {
        return sessions.get(sessionId);
      } finally {
        // 1000개의 Usersession 객체를 UserSessions에 추가한다
        int sessionCount = 1_000;
        for (int i = 0; i <= sessionCount; i++) {
          String sessionId = "session-" + i;
          Future<?> future = executor.submit(() -> { // 500개 스레드를 이용해 동시에 실행
            UserSession userSession = new UserSession(sessionId);
            userSessions.addUserSession(userSession);
          });
          futures.add(future);
        }
        // 1000개의 작업이 끝날 때까지 대기
        futures.forEach(f -> {
          try {
            f.get();
          } catch (Exception e) {
            log.error("error", e);
          }
        });

        executor.shutDown();

        // 검증
        for(int i = 1; i <= sessionCount; i++) {
          String sessionId = "session-" + i;
          UserSession userSession = userSessions.getUserSession(sessionId);
          assertThat(userSession)
            .describeAs("session %s", sessionId);
            .isNotNull();
        }
      }
    }
  }
  ```
- 만약 lock 없이 sessions.put()를 실행하면 누락되는 객체가 생겨 테스트 
- ReentrantLock은 한 번에 한 스레드만 lock을 구할 수 있음
  - 나머지 스레드는 대기 필요
> mutex는 mutual exclusion의 줄임말로 lock이라 하기도 함
> java는 Lock 타입을 사용하고 Go는 Mutex 타입을 사용
### 세마포어
- 동시에 실행할 수 있는 스레드 수를 제한
- 자원에 대한 접근을 일정 수준으로 제한하고 싶을 때 사용
- 허용 가능한 숫자를 이용하여 생성
  - 이 숫자를 java에서는 permit, Go에서는 weight이라 함

