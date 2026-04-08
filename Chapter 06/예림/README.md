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
- 세마포어를 사용하는 전형적인 순서
  - 세마포어에서 퍼밋 획득 (허용 가능 숫자 1 감소)
  - 코드 실행
  - 세마포어에 퍼밋 반환 (허용 가능 숫자 1 증가)
- 퍼밋 개수가 0이면 대기
> 세마포어에서 퍼밋을 구하고 반환하는 연산을 각각 P(wait) 연산, V(signal) 연산이라고 함

- 코드 예시
  ```java
  public class MyClient {
    private Semaphore semaphore = new Semaphore(5);

    public String getData() {
      try {
        semaphore.acquire(); // 퍼밋 획득 시도
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
      try {
        String data = ...; // 외부 연동 코드
        return data;
      } finally {
        semaphore.release(); // 퍼밋 반환
      }
    }
  }
  ```
- 읽기 쓰기 잠금 특징
  - 쓰기 잠금은 한 번에 한 스레드만
  - 읽기 잠금은 한 번에 여러 스레드 가능
  - 한 스레드가 쓰기 잠금을 획득하면 쓰기 잠금 해제될 때까지 읽기 잠금 획득 불가
  - 읽기 잠금을 획득한 모든 스레드가 읽기 잠금 해제할 때까지 쓰기 잠금 획득 불가
- 코드 예시
  ```java
  public class UserSessionRw {
    private ReadWriteLock lock = new ReentrantReadWriteLock();
    private Lock writeLock = lock.writeLock();
    private Lock readLock = lock.readLock();
    private Map<String, UserSession> sessions = new HashMap<>();

    public void addUserSession(UserSession session) {
      writeLock.lock(); 
      try {
        sessions.put(session.getSessionId(), session); // 공유 자원 접근
      } finally {
        writeLock.unlock();
      }
    }
    
    public UserSession getUserSession(String sessionId) {
      readLock.lock();
      try {
        return sessions.get(sessionId);
      } finally {
        return readLick.unlock();
      }
    }
  }
  ```
### 원자적 타입
- 스레드가 동시에 데이터를 변경할 때 발생하는 문제를 잠금을 활용하면 다음과 같이 해결할 수 있다
- lock 사용 전
  ```java
  public class Increaser {
    private int count = 0;

    public void inc() {
      count = count + 1;
    }
  }
  ```
- lock 사용 후
  ```java
  public class Increaser {
    // lock 선언
    private int count = 0;

    public void inc() {
      lock.lock();
      try {
        count = count + 1;
      } finally {
        lock.unlock();
      }
    }
  }
  ```
- 하지만 이렇게 하면 CPU 효율이 떨어짐

- lock을 사용하지 않으면서 동시성 문제 없이 카운터 구현하는 방법 : 원자적 타입
- 자바 언어로는 AtomicInteger, AtomicLong, AtomicBoolean
```java
public class Increaser {
  private AtomicInteger count = new AtomicInteger(0);

  public void inc() {
    count.incrementCountAndGet(); // 다중 스레드 문제 없이 값을 1 증가 시킴
  }
}
```
- AtomicInteger은 내부적으로 CAS 연산 사용
  - Compare and Swap
- 내부 구현은 복잡하지만 lock보다 간단하게 동시성 문제 해결
### 동시성 지원 컬렉션
- 데이터를 변경하는 모든 연산에 lock 적용하도록 제한
- 자바 Collections 클래스는 동기화된 컬렉션을 생성하는 메서드 제공

```java
Map<String, String> map = new HashMap<>();
// 동기화된 컬렉션 객체 생성
Map<String, String> syncMap = Collections.syncronizedMap(map);
syncMap.put("key1", "value1"); // put 메서드는 내부적으로 synchronized로 처리됨

```
- 또 다른 방법 : 동시성 자체를 지원하는 컬렉션 타입 사욜

```java
ConcurrentMap<String, String> map = new ConcurrentMap<>();
map.put("key1", "value1"); // 동시성 지원 컬렉션은 동시성 지원 범위를 최소화한다
```

## DB와 동시성
- 대부분의 DB는 명시적인 잠금 기법 제공 -> 선점적(비관적) 락
- 비관적 락을 사용하면 동일한 레코드에 한 번에 하나의 트랜잭션만 접근할 수 있도록 제어
  - 값 비교 후 수정은 낙관적 락

### 비관적(선점) 락
- 데이터에 먼저 접근한 트랜잭션이 락 획득


```sql
select * from 테이블 where 조건
for update
```
- 레코드를 조회하면서 락을 획득하고 트랜잭션 종료 시 반환

### 낙관적(비선점) 락
- 데이터를 조회한 시점의 값과 수정한 시점의 값이 같은지 비교히는 방식
- 정수 타입의 버전 칼럼 사용

1. select 쿼리 실행 시 version 칼럼 함께 조회
  ```sql
  select ..., version from 테이블 where id = 아이디
  ```
2. 로직 수행
3. update 쿼리 수행 시 version 컬럼 1씩 증가
  - 이때 version 값이 1번에서 조회한 값과 같은지 비교하는 조건을 where 절에 추가
    ```sql
    UPDATE 테이블 SET …, version = version + 1
    WHERE id = 아이디 AND version = [1에서 조회한 version 값]
    ```
4. update 쿼리 결과로 변경된 행 개수가 0이면 이미 다른 트랜잭션이 수정한 것이므로 데이터 변경 실패한 것이니 롤백
5. update 결과로 변경된 행 개수가 0보다 크면 트랜잭션 커밋

### 외부 연동과 잠금
- 외부 시스템 연동 시 비관적 락(선전 잠금) 추천
- 결제는 이미 취소 됐는데 롤백이 발생하는 경우가 생길 수 있음
- 낙관적 락 사용하고 싶다면 트랜잭션 아웃박스 패턴 사용 추천

### 증분 쿼리
- 잠금을 사용하지 않으면서 참여자 수를 증가시키는 방법

```sql
update SUBJECT set joinCount = joinCount + 1 where id = ?
```
- DB는 이를 원자적 연산으로 처리

## 잠금 사용 시 주의 사항
### 잠금 해제하기
- 잠금을 획득한 후에는 꼭 해제하기 (무한 대기 방지)
- 습관적으로 try-finally 구조를 사용해라

### 대기 시간 지정하기
- 잠금 획득할 때까지 대기해야 하므로 동시 접근 요청이 많이 오면 대기 시간이 길어짐
- 해결 방법 중 하나는 **대기 시간을 지정**하는 것
```java
boolean acquired = lock.tryLock(5, TimeUnit.SECONDS);
if(!acquired) {
  throw new RunTimeException("Failed to acquire lock");
}
// 잠금 획득 성공
try {
  // 자원 접근 코드 실행
} finally {
  lock.unlock();
}
```
- 대기 시간 없이 반환하고 싶으면
```java
lock.tryLock();
```
### 교착 상태(deadlock) 피하기
한 자원에서 다른 자원으로 용량을 전송하는 코드를 구현한다고 할 때, 다음과 같이 두 자원의 잠금이 필요
```
자원 A 잠금 획득
자원 B 잠금 획득
자원 A 용량 감소
자원 B 용량 증가
자원 A 잠금 해제
자원 B 잠금 해제
```
- 이렇게 한 작업에서 두 번의 잠금을 획득하는 구조는 deadlock 상태에 빠지기 쉬운 전형적인 패턴
- deadlock 방지
  - 잠금 대기 시간을 제한
  - 지정한 순서대로 잠금 획득

## 단일 스레드로 실행하기
- 한 스레드만 자원에 접근하도록 하면 동시성 문제를 피할 수 있다
- 각 스래드는 작업 큐에 필요한 작업만 추가할 뿐 직접 상태에 접근하지 못하고 상태 관리 스레드만 큐에서 작업을 꺼내 데이터 처리 수행

```java
while (running) {
  // 한 스레드만 작업 큐에서 꺼내 실행
  Job job = jopQueue.poll(1, TimeUnit.SECONDS);
  if (job == null) {
    continue;
  }
  // job 종류에 따라 상태 처리
  while (job.getType()) {
    case INC:
      // modifyState()는 한 스레드만 접근하므로 상관 없다
      obj.modifyState();
      break(
    // ... 다른 작업
    }
    // ...
  }
```
- 두 스레드가 자원 공유가 필요하면 콜백이나 큐를 이용해 **데이터 복제본**이나 **불변 값**을 공유하기도
  - 다른 스레드에서 이를 수정하지 못하게 해서 동시성 문제 방지
- 단일 스레드 단점이 있다면 구조가 복잡해짐
- 논블로킹과 비동기 IO에선 블로킹 연산을 최소화해야 하므로 단일 스레드 방식이 적합

- 성능은 동시 접근하는 스레드가 적을수록 큐나 채널을 처리하지 않아도 되는 모잠금이 더 좋지만 동시접근 스레드가 많아질수록 단일 스레드 방식이 더 나은 성능을 낼 수도 있음

