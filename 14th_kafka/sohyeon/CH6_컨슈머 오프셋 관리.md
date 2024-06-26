# 6장 컨슈머 오프셋 관리
- 컨슈머의 주요 역할은 **카프카에 저장된 메시지를 가져오는 것**이다.

<br/>

## 6.1 컨슈머 오프셋 관리
- 컨슈머가 **메시지를 어디까지 가져왔는지 표시**하는 것은 중요하다.
- 오프셋(offset)
  - 메시지의 위치를 나타내는 위치 (숫자 형태)
  - `_consumer_offsets` 토픽에 각 컨슈머 그룹별로 위치 정보가 기록된다.

<br/>

<img width="700" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/fe56cee7-f6d7-47ed-9f38-478daf4905bf">

🔼 컨슈머 기본 동작
- 컨슈머들은 지정된 토픽의 메시지를 읽은 뒤, 읽어온 위치와 오프셋 정보를 `_consumer_offsets`에 기록한다.
  - 오프셋값: 컨슈머가 다음으로 읽어야 할 위치
  - 1️⃣ 2번 오프셋 메시지 C까지 읽었다.
  - 2️⃣ 그 다음으로 읽어야 할 3번 오프셋 위치를 저장한다.
- 모든 컨슈머 그룹의 정보가 저장되는 `_consumer_offsets` 토픽은 파티션 수와 리플리케이션 팩터 수를 가진다.
  - offsets.topic.num.partitions: 기본값 50
  - offsets.topic.replication.factor: 기본값 3

<br/>

## 6.2 그룹 코디네이터
- 컨슈머 그룹 내의 각 컨슈머들은 서로 자신의 정보를 공유하면서 하나의 공동체로 동작한다.
- 컨슈머 그룹은 각 컨슈머들에게 작업을 균등하게 분배해야 한다. (컨슈머 리밸런싱 consumer rebalancing)
- 안정적인 컨슈머 그룹 관리를 위해 별도의 **코디네이터**가 존재한다. (그룹 코디네이터)
  - 컨슈머 그룹이 구독한 토픽의 파티션들과 그룹의 멤버들을 트래킹힌다.
  - 카프카 클러스터 내의 브로커 중 하나에 위치한다.
 
<br/>

<img width="500" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/43e9edd1-d508-4fb1-b303-c7ad6755ee22">

🔼 그룹 코디네이터의 컨슈머 그룹 관계
- 컨슈머 그룹이 브로커에 최초 연결 요청을 보내면 브로커 중 하나에 그룹 코디네이터가 생성된다.
- 그룹 코디네이터는 컨슈머 그룹의 컨슈머 변경과 구독하는 토픽 파티션 등에 대한 감지를 시작한다.
- 토픽의 파티션과 그룹의 멤버 변경이 일어나면 변경된 내용을 컨슈머들에게 알린다.

<br/>

<img width="811" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/80fd7f44-060c-445a-98ac-3b8e01f64ff0">

🔼 컨슈머 그룹 등록 과정
1. 컨슈머는 컨슈머 설정값 중에서 bootstrap.brokers 리스트에 있는 브로커에게 컨슈머 클라이언트와 초기 커넥션을 연결하기 위한 요청을 보낸다.
2. 해당 요청을 받은 브로커는 그룹 코디네이터를 생성하고 컨슈머에게 응답을 보낸다. (컨슈머 등록 전까지 아무 작업도 일어나지 않음)
3. 그룹 코디네이터는 group.initial.rebalance.delay.ms의 시간 동안 컨슈머의 요청을 기다린다.
4. 컨슈머는 컨슈머 등록 요청을 그룹 코디네이터에게 보낸다. 이때 가장 먼저 요청을 보내는 컨슈머가 컨슈머 그룹의 리더가 된다.
5. 컨슈머 등록 요청을 받은 그룹 코디네이터는 해당 컨슈머 그룹이 구독하는 토픽 파티션 리스트 등 리더 컨슈머의 요청에 응답을 보낸다.
6. 리더 컨슈머는 정해진 컨슈머 파티션 할당 전략에 따라 그룹 내 컨슈머들에게 파티션을 할당한 뒤 그룹 코디네이터에게 전달한다.
7. 그룹 코디네이터는 해당 정보를 캐시하고 각 그룹 내 컨슈머들에게 성공을 알린다.
8. 각 컨슈머들은 각자 지정된 토픽 파티션으로부터 메시지들을 가져온다.

<br/>

|컨슈머 옵션|값|설명|
|---|---|---|
|heartbeat.interval.ms|3000|- 기본값은 3000이며<br/>- 그룹 코디네이터와 하트비트 인터벌 시간<br/>- session.timeout.ms의 1/3이 적당하다.|
|session.timeout.ms|10000|- 기본값 10000<br/>- 어떤 컨슈머가 특정 시간 안에 하트비트를 받지 못하면 문제가 발생한다고 판단해 컨슈머 그룹에서 해당 컨슈머는 제거되고 리밸런싱 동작이 일어난다.|
|max.poll.interval.ms|300000|- 기본값 300000<br/>- 컨슈머는 주기적으로 poll()을 호출해 토픽으로부터 레코드를 가져오는데, 호출 후 최대 5분간 pll() 호출이 없다면 컨슈머가 문제 있는 것으로 판단해 리밸런싱 동작이 일어난다.|

🔼 컨슈머 하트비트 옵션
- 하트비트를 주기적으로 주고받으면서 그룹 코디네이터는 컨슈머가 잘 살아있는지, 잘 동작하는지 확인한다.
- 할당된 파티션에서 컨슈머가 정상적으로 메시지를 가져가고 있는지 poll() 동작 여부를 통해 확인할 수도 있다.
- 컨슈머의 다운을 너무 빠르게 감지하면 원치 않은 리밸런싱이 빈번하게 발생할 수 있고, 너무 느리게 감지하면 파티션의 메시지를 읽지 못하는 현상이 발생할 수 있다.

<br/>

## 6.3 스태틱 멤버십
- 컨슈머의 설정 변경이나 소프트웨어 업데이트로 인해 컨슈머가 재시작되면, 컨슈머 그룹 내의 동일한 컨슈머임에도 새로운 컨슈머로 인식해 새로운 엔티티 ID가 부여되고 이로 인해 컨슈머 그룹의 리밸런싱이 발생한다.
- 불필요한 리밸런싱을 방어하기 위해 아파치 카프카 2.3부터 **스태틱 멤버십(static membership)** 이라는 개념을 도입되었다.
- 컨슈머마다 인식할 수 있는 ID를 적용함으로써 다시 합류하더라도 그룹 코디네이터가 기존 구성원임을 인식할 수 있게 한다.
- 스태틱 멤버십 기능을 적용한다면 session.timout.ms를 기본값보다는 큰 값으로 조정해야 한다. (하트비트 이슈를 피하기 위함)

<br/>

### 실습 (일반)
- 파이썬3 설치 (파이썬 가상 환경 이용, 파이썬3으로 작성된 컨슈머를 사용하기 위함)
- 토픽 생성 (peter-test06, 파티션 수 3, 리플리케이션 팩터 수 3)
- 모든 브로커에 접속하여 깃허브를 클론한 후 `consumer_standard.py` 실행

<br/>

```
print(
  'Topic: {}, '
  'Partition: {}, '
  'Offset: {}, '
  'Received message: {}'.format(
                            msg.topic(),
                            msg.partition(),
                            msg.offset(),
                            msg.value().decode('utf-8')))
```
🔼 consumer_standard.py 내용 중 변경 부분
- 출력 내용을 확인하기 위해 파일 내용 일부 변경

<br/>

```
some_data_source = []
for messageCount in range(1, 11):
  some_data_source.append('Apache Kafka is a distributed streaming platform - %d' % messageCount)
```
🔼 producer.py 내용 중 변경 부분
- 총 10개의 메시지를 보낸다.

<br/>

```
(venv6) $ python producer.py
```
🔼 프로듀서 실행

<br/>

![image](https://github.com/mash-up-kr/S3A/assets/55437339/d4d5319b-6675-4275-882d-9ba012144371)

🔼 프로듀서 실행 후 컨슈머에서 메시지를 읽은 내용
- 각 컨슈머들이 메시지를 정상적으로 읽었다.

<br/>

```
$ /usr/local/kafka/bin/kafka-consumer-groups.sh --bootstrap-server peter-kafka01.foo.bar:9092 --group peter-consumer01 --describe
```

<br/>

![image](https://github.com/mash-up-kr/S3A/assets/55437339/2653705e-b8a0-42f7-af16-4fe99e7398ab)

🔼 해당 컨슈머 그룹의 상세보기 확인
- 파티션별로 LAG 없이 모든 메시지를 읽어온 것을 확인할 수 있다.

<br/>

<img width="688" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/5a860d95-e473-47f6-872c-2a30c11cf2d1">

🔼 일반 컨슈머 그룹의 리밸런싱 동작(1)
- 현재 peter-test06 토픽의 파티션이 컨슈머의 프로세스가 실행된 브로커와 매핑된 내용을 확인할 수 있다.
- peter-kafka01의 컨슈머 프로세스를 강제로 종료한다.

<br/>

![image](https://github.com/mash-up-kr/S3A/assets/55437339/fd22167c-0b49-42ce-a434-2cdbff22617d)

🔼 컨슈머 프로세스 종료 후 그룹 상세보기
- 그룹 코디네이터는 상황을 인지하고 그룹 내부적으로 리밸런싱이 일어났다.

<br/>

<img width="685" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/3a9f9c52-e7c5-4226-906f-e4435fd82cf1">

🔼 일반 컨슈머 그룹의 리밸런싱 동작(2)
- 현재 peter-test06 토픽의 파티션이 컨슈머의 프로세스가 실행된 브로커와 매핑된 내용을 확인할 수 있다.
- 브로커 peter-kafka02의 컨슈머가 담당하고 있던 파티션도 변경됐다. (전체 컨슈머를 대상으로 리밸런싱 동작)
- 불필요한 리밸런싱은 최대한 줄여야 한다.

<br/>

### 실습 (스태틱 멤버십)
```
hostname = socket.gethostname()
broker = 'peter-kafka01.foo.bar'
group = 'peter-consumer02'
topic = 'peter-test06'

c = Consumer({
  'bootstrap.servers': broker,
  'group.id': group,
  'session.timeout.ms': 30000,
  'group.instance.id': 'consumer-' + hostname,
  'auto.offset.reset': 'earliest'
})
```
🔼 스태틱 컨슈머 설정

<br/>

```
(venv6) $ python consumer_static.py
```
🔼 모든 브로커에서 스태틱 멤버십 설정이 적용된 컨슈머를 실행

<br/>

<img width="684" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/bca28cf6-585f-4247-b753-175f4b4b33a9">

🔼 스태틱 멤버십이 적용된 컨슈머 그룹의 리밸런싱 동작(1)
- 현재 peter-test06 토픽의 파티션이 컨슈머의 프로세스가 실행된 브로커와 매핑된 내용을 알 수 있다.
- 실행 중인 컨슈머 프로세스를 강제로 종료한다.
  - 종료 후에도 리밸런싱 동작이 일어나지 않는다.
  - session.timeout.ms 경과 후 리밸런싱 동작 후 새로운 파티션이 할당된다.

<br/>

<img width="680" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/191c04e7-0b36-4212-a399-d0ccadbc930c">

🔼 스태틱 멤버십이 적용된 컨슈머 그룹의 리밸런싱 동작(2)
- 리밸런싱 동작에 의해 변경된 내용을 확인할 수 있다.
- 프로세스 종료 전 peter-kafka01 컨슈머가 담당한 파티션 1번이 peter-kafka02 담당으로 변경됐다.
- 종료된 컨슈머만 제거되고, 담당하던 파티션을 재담당함으로써 총 2번의 리밸런싱 동작이 발생했다.

<br/>

## 6.4 컨슈머 파티션 할당 전략
컨슈머 그룹의 리더 컨슈머가 정해진 파티션 할당 전략에 따라 각 컨슈머와 대상 토픽의 파티션을 매칭시킨다.

<br/>

|파티션 할당 전략|설명|
|---|---|
|레인지 파티션 할당 전략|- 파티션 할당 전략의 기본값으로서 토픽별로 할당 전략을 사용한다.<br/>- 동일한 키를 이용하는 2개 이상의 토픽을 컨슘할 때 유용하다.|
|라운드 로빈 파티션 할당 전략|- 사용 가능한 파티션과 컨슈머들을 라운드 로빈으로 할당한다. <br/>- 균등한 분배가 가능하다.|
|스티키 파티션 할당 전략|- 컨슈머가 컨슘하고 있는 파티션을 계속 유지할 수 있다.|
|협력적 스티키 파티션 할당 전략|- 스티키 방식과 유사하지만, 전체 일시 정지가 아닌 연속적인 재조정 방식이다.|

🔼 파티션 할당 전략

<br/>

### 6.4.1 레인지 파티션 할당 전략
- 먼저 구독하는 토픽에 대한 파티션을 순서대로 나열한 후 컨슈머를 순서대로 정렬한다.
- 그런 다음 각 컨슈머가 몇 개의 파티션을 할당해야 하는지 전체 파티션 수를 컨슈머 수로 나눈다.

<br/>

<img width="518" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/19f1d01e-1fe5-48b1-ba3d-0d2e00a4997f">

🔼 레인지 파티션 할당 전략
- 파티션을 0,1,2 순서대로 정렬한다.
- 컨슈머를 1,2 순서대로 정렬한다.
- 전체 파티션 수(3)를 전체 컨슈머 수(2)로 나눈다. (컨슈머당 최소 1개의 파티션을 가짐)
- 남은 파티션은 먼저 정렬된 컨슈머에 할당한다. (👎 불균형 할당)
- 👍 동일한 레코드 키를 사용하고 하나의 컨슈머 그룹이 동일한 파티션 수를 가진 2개 이상의 토픽을 컨슘할 때 유용하다.

<br/>

### 6.4.2 라운드 로빈 파티션 할당 전략
- 먼저 컨슘해야 하는 모든 파티션과 컨슈머 그룹 내 모든 컨슈머들을 나열한다.
- 라운드 로빈으로 하나씩 파티션과 컨슈머를 할당한다.

<br/>

<img width="537" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/6d8dac93-2943-4148-96b5-f5f8e3baaa6e">

🔼 라운드 로빈 파티션 할당 전략
- 먼저 구독 대상 토픽의 전체 파티션을 나열한다.
- 파티션 나열 후 전체 컨슈머들도 나열한다.
- 나열된 파티션을 컨슈머들과 하나씩 라운드 로빈 방식으로 일대일 매핑해나간다.

<br/>

|파티션|매핑된 컨슈머|
|---|---|
|토픽1-파티션0|컨슈머1|
|토픽1-파티션1|컨슈머2|
|토픽1-파티션2|컨슈머1|
|토픽2-파티션0|컨슈머2|
|토픽2-파티션1|컨슈머1|
|토픽2-파티션2|컨슈머2|

🔼 라운드 로빈 방식으로 매핑된 파티션과 컨슈머
- 하나씩 번갈아 가면서 컨슈머를 할당한다.
- 👍 균등하게 매핑할 수 있다.

<br/>

### 6.4.3 스티키 파티션 할당 전략
- 기존에 매핑됐던 파티션과 컨슈머를 최대한 유지하려고 하는 전략
- 가능한 한 균형 잡힌 파티션을 할당한다.
- 재할당이 발생할 때 되도록 기존의 할당된 파티션 정보를 보장한다. (일부 파티션은 새로운 컨슈머와 연결될 수 있음)

<br/>

<img width="666" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/217218bd-4c72-4cd0-90ce-1d0b72b6b161">

🔼 최초 스티키 파티션 할당 전략

<br/>

<img width="526" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/ddfb154e-b1f9-451d-b15f-f6ca4b5e5876">

🔼 라운드 로빈 파티션 할당 전략에서 컨슈머2가 제외된 상황
1. 컨슈머2가 컨슈머 그룹에서 떠남
2. 리밸런싱 동작이 일어남
3. 모든 파티션을 순서대로 배치
4. 모든 컨슈머들을 순서대로 배치
5. 라운드 로빈 파티션 할당 전략에 맞춰 하나씩 매핑
- 일하던 컨슈머1과 컨슈머3에 할당됐던 파티션 연결은 끊기고 리밸런싱으로 인해 새로운 파티션이 할당된다.

<br/>

<img width="596" alt="image" src="https://github.com/mash-up-kr/S3A/assets/55437339/f9740684-c8dc-4c0e-8654-c7d45058ac29">

🔼 스티키 파티션 할당 전략에서 컨슈머 2가 제외된 상황
- 기존 컨슈머1과 컨슈머3에 할당됐던 파티션들은 모두 유지한 채, 컨슈머2에 할당된 파티션들만 새로 할당된다.
- 컨슈머들의 최대 할당된 파티션의 수의 차이는 1
- 기존에 존재하는 파티션 할당은 최대한 유지함
- 재할당 동작 시 유효하지 않은 코든 파티션 할당은 제거함
- 할당되지 않은 파티션들은 균형을 맞추는 방법으로 컨슈머들에 할당
- 👍 컨슈머에 새로 할당하는 파티션 수를 최소화한다. (라운드 로빈 할당 전략보다 효율적)

<br/>

### 6.4.4 협력적 스티키 파티션 할당 전략
- 리밸런싱이 일어나도 기존의 컨슈머와 파티션 매핑은 유지하고 최소한의 파티션만 컨슈머와 매핑한다.
- COOPERATIVE 프로토콜(내부 리밸런싱 프로토콜)을 적용한다. (리밸런싱 동작 전 컨슈머 상태 유지 가능)
- 되도록 동작 중인 컨슈머들에게 영향을 주지 않는 상태에서 몇 차례에 걸쳐 리밸런싱이 이뤄진다.

<br/>

![image](https://github.com/mash-up-kr/S3A/assets/55437339/6833f5e0-e105-4968-b694-69f037288d60)

🔼 일반적인 리밸런싱 동작 과정
- `감지`: peter-kafka02의 컨슈머가 다운됨을 감지
- `중지`: 컨슈머에 할당된 모든 파티션을 제거 (LAG 증가)
- `재시작`: 구독한 파티션이 컨슈머들에게 재할당 (다운타임 종료)
- 👍 한 번에 모든 파티션 할당 작업이 끝난다.
- 👎 전체 컨슈머가 일시적으로 멈춘 상태에서 리밸런싱이 이뤄진다.

<br/>

![image](https://github.com/mash-up-kr/S3A/assets/55437339/8094a309-a73e-4de6-bf98-5d64dbcf5dc0)

🔼 협력적 스티키 리밸런싱 동작 과정
1. 컨슈머 그룹에 peter-kafka01이 합류하면서 리밸런싱이 트리거된다.
2. 컨슈머 그룹 내 컨슈머들은 그룹 합류 요청과 자신들이 컨슘하는 토픽의 파티션 정보(소유권)를 그룹 코디네이터로 전송한다.
3. 그룹 코디네이터는 해당 정보를 조합해 컨슈머 그룹의 리더에게 전송한다.
4. 컨슈머 그룹의 리더는 현재 컨슈머들이 소유한 파티션 정보를 활용해 제외해야 할 파티션 정보를 담은 새로운 파티션 할당 정보를 그룹 멤버들에게 전달한다.
5. 새로운 파티션 할당 정보를 받은 컨슈머 그룹 멤버들은 현재의 파티션 할당 전략과 차이를 비교해보고 필요 없는 파티션을 골라 제외한다. 이전의 파티션 할당 정보와 새로운 파티션 할당 정보가 동일한 파티션들에 대해서는 어떤 작업도 수행할 필요가 없다.
6. 제외된 파티션 할당을 위해 컨슈머들은 다시 합류 요청을 한다. 여기서 두번째 리밸런싱이 트리거된다.
7. 컨슈머 그룹의 리더는 제외된 파티션을 적절한 컨슈머에게 할당한다.

<br/>

## 6.5 정확히 한 번 컨슈머 동작
```java
import java.util.List;
import java.util.Properties;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;

public class ExactlyOnceConsumer {
	public static void main(String[] args) {
		String bootstrapServers = "peter-kafka01.foo.bar:9092";
		Properties props = new Properties();
		props.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
		props.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
		props.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
		props.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "peter-consumer-01");
		props.setProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
		props.setProperty(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
		props.setProperty(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed"); // 정확히 한 번 전송을 위한 설정

		KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
		consumer.subscribe(List.of("peter-test05"));
		
		try {
			while (true) {
				ConsumerRecords<String, String> records = consumer.poll(1000);
				for (ConsumerRecord<String, String> record : records) {
					System.out.printf("Topic: %s, Partition: %s, Offset: %d, Key: %s, Value: %s\n"
							, record.topic()
							, record.partition()
							, record.offset()
							, record.key()
							, record.value()
					);
				}
				consumer.commitAsync();
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			consumer.close();
		}
	}
}
```
🔼 트랜잭션 컨슈머 예제 코드 (ExactlyOnceConsumer.java)
- 일반 컨슈머 코드에서 `ISOLATION_LEVEL_CONFIG` 설정만 추가하면 트랜잭션 컨슈머로 동작한다.
- 트랜잭션 컨슈머는 트랜잭션이 완료된 메시지만 읽을 수 있다. (트랜잭션 코디네이터와 통신 X)

<br/>

### 실습
```
$ java -jar ExactlyOnceConsumer.jar
```
🔼 JAR 파일을 이용한 컨슈머 실행

<br/>

```
$ java -jar ExactlyOnceProducer.jar
```
🔼 프로듀서 실행
- 메시지를 전송한다.

<br/>

```
Topic: peter-test05, Partition: 0, Offset: 2, Key: null, Value: Apache Kafka is a distributed streaming platform - 0
```
🔼 출력 결과
- 한 건의 메시지를 전송했는데 오프셋이 2로 표시된다.
- 트랜잭션의 종료를 표시하기 위해 메시지 전송 후 커밋 또는 중단에 대한 표시를 남기는 트랜잭션 메시지가 추가되기 때문이다.

<br/>

### 트랜잭션 동작
- 컨슈머는 트랜잭션 코디네이터와 통신하는 부분이 없으므로 **정확하게 메시지를 한 번 가져오는지는 보장할 수 없다.**
- 컨슈머에 의해 컨슘된 메시지가 다른 싱크 저장소로 중복 저장될 수도 있다.
- 정확히 한 번 처리가 가능하려면 `컨슘 - 메시지 처리 - 프로듀싱` 동작이 모두 하나의 트랜잭션으로 처리되어야 한다.
