# 8. GC 로깅, 모니터링, 튜닝, 툴

### 8.1 GC 로깅 개요
- GC 로그는 중요하다
- 문제가 생긴 지점을 알 수 있음
- GC 로깅은 사실상 공짜이고 유용하다

#### 8.1.1 GC 로깅 켜기

- JVM 옵션으로 키면 된다.
- JVM 옵션같은건 버전마다 구현체마다 다를 수 있어서 그대그때 찾아보는게 좋을듯?
- 파일경로, 이벤트세부정보(verbose 할지?), 시간을 출력

#### 8.1.2 GC 로그 vs JMX
- GC 로그는 GC 가 발생할때마다
- JMX는 샘플링해서 데이터를 얻음
- JMX가 당연히 더 데이터가 많지만, 성능 하락
- 근데 실제적으로 몇퍼 안될듯

#### 8.1.3 JMX 의 단점
- 샘플링의 단점들
- 보안 문제
- 핵심 포인트를 놓치기 쉬움

#### 8.1.4 GC 로그 데이터의 장점
- GC 로그는 특정 GC와 연관지을 수 있어서
- 애플리케이션의 중요한 포인트를 정확히 파악할 수 있다.

### 8.2 로그 파싱 툴
- 로그 포맷도 뭐쓰느냐에 따라 제각기~

#### 8.2.1 센섬
- 생략

### 8.2.2 GCViewer
- 생략

## 8.3 GC 기본 튜닝
- GC 가 저렴하고 제일 효율적이다.
- 튜닝 포인트
    - 할당
    - 중단 민감도
    - 처리율 추이
    - 객체 수명

- Xms, Xmx
- XX:MaxMetaspaceSize
- GC 튜닝에 대해 주저리 주저리 나오는데,,, GC는 건들지 않고 버전업만 잘하라는 시니어분의 의견

### 8.3.1 할당이란
- 메모리를 적게 할당하자 -> 다른 JVM책에선 이것만 얘기함.
- MaxTenuringThreshold : 몇번 돌아야 테뉴어드로 가는지


### 8.3.2 중단 시간이란?
- g1gc가 중단시간 작게 가져가니 좋다는 말들..

### 8.3.3 수집기 스레드와 gc 루트


## 8.4 Parallel GC 튜닝
- 건들지 마라

## 8.5 CMS 튜닝
- 얘도 건들지 마라.


## 8.6 g1 튜닝
- g1 GC를 java7 이었던 시절에 쓴거라... 책 내용이랑은 많이 다를 수 있음

