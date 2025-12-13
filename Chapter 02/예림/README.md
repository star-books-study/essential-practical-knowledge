# 2장. 느려진 서비스, 어디부터 봐야 할까

- 다양한 지표가 성능과 관련 -> 네트워크 속도, 디스크 속도, 메모리 크기, 디바이스의 CPU 속도
- 서버 성능과 관련된 중요한 지표 -> 응답 시간과 처리량

## 응답 시간
- 응답 시간 : 사용자의 요청을 처리하는 데 걸리는 시간
- 클라이언트가 서버로 요청을 보내는 과정은 크게 2단계로 이루어진다.
  <img width="595" height="247" alt="image" src="https://github.com/user-attachments/assets/23206c65-eeaf-4bb9-b1f4-62bac81a7329" />
- 하나의 API 요청을 처리하는 데 걸리는 시간은 다음과 같이 구성된다.
  <img width="578" height="166" alt="image" src="https://github.com/user-attachments/assets/e20a4ce3-9159-4f16-a766-4af7292d01ce" />
- 응답 시간은 다음과 같이 2가지로 나누어 측정하기도 한다.
  - TTFB(Time to First Byte): 응답 데이터 중 첫 번째 바이트가 도착할 때까지 걸린 시간
  - TTLB(Time to Last Byte): 응답 데이터의 마지막 바이트가 도착할 때까지 걸린 시간
