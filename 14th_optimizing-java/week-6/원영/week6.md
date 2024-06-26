# 6. 가비지 수집 기초
- 처음에는 메모리를 관리할 수 없어서 반감을 샀지만, 지금은 짱굿나이스
- 요체: 객체의 수명을 정확히 몰라도 런타임이 대신 객체를 추적하며 쓸모없는 객체를 알아서 제거하는 것
- 기본 원칙
    - 알고리즘은 반드시 모든 가비지를 수집해야 한다.
    - 살아 있는 개체는 절대로 수집해선 안 된다. (중요)

## 6.1 마크 앤 스위프

- 전체적인 GC 알고리즘
    - 할당 리스트를 순회하면서 마크 비트를 지운다.
    - GC 루트부터 살아 있는 객체를 찾는다.
    - 이렇게 찾은 객체마다 마크 비트를 세팅한다.
    - 할당 리스트를 순회하면서 마크 비트가 세팅되지 않은 객체를 찾는다.
        - 힙에서 메모리를 회수해 프리 리스트에 되돌린다.
        - 할당 리스트에서 객체를 삭제한다.
- 살아 있는 객체는 대부분 DFS 방식 → 생성된 그래프 = live object graph

![image](https://github.com/mash-up-kr/S3A/assets/74983448/2c0eaa35-4f94-449b-b113-8581a5305561)
### 6.1.1 가비집 수집 용어

- STW: GC 사이클이 발생하여 가비지를 수집하는 동안 GC
- 동시: GC 스레드는 애플리케이션 스레드와 동시 실행 가능, 계산 비용이 큼, 100% 보장 x
- 병렬: 여러 스레드를 동원해서 가비지 수집
- 정확: 정확한 GC 스킴은 전체 가비지를 한방에 수집할 수 있게 힙 상태에 관한 타입 정보를 지니고 있음
- 보수: 보수적인 스킴은 정확한 스킴의 정보 x, 리소스를 낭비하는 일이 잦고, 근본적으로 타입 체계 무시
- 이동: 객체는 이동 수집기에서 메모리를 오갈 수 있음 = 객체 주소 고정 X
- 압착: GC 사이크 마지막에 allocated memory 들은 단일 영역으로 (대개 이 영역 첫 부분) 배열
- 방출: GC cycle 에서 마지막에 할당된 영역을 비우고 살아남은 객체를 다른 memory 영역으로 방출

## 6.2 핫스팟 런타임 개요

- 자바 언어 특징
    - 기본형, 객체 레퍼런스 두 가지 값만 사용
    - C++와 달리 주소를 역참조하는 일반적인 메커니즘 x
    - 오직 오프셋 연산자만으로 필드에 액세스 || 객체 레퍼런스의 메서드 호출
    - 값으로 호출 방식으로만 메서드 호출
    

### 6.2.1 객체를 런타임에 표현하는 방법

**oop(ordinary object pointer)**

- 평범한 객체 포인터
- 참조형 지역 변수 안에 위치
- 자바 메서드의 스택 프레임으로부터 자바 힙을 구성하는 메모리 영역 내부를 가르킴
- **instanceOop**: oop를 구성하는 자료 구조 중 하나
    - 자바 클래스의 인스턴스
    - 메모리 레이아웃은 모든 객체에 대해 기계어 워드 2개로 구성된 헤더
        - Mark 워드: 인스턴스 관련 메타데이터를 가리키는 포인터
        - Klass 워드: 클래스 메타데이터를 가리키는 포인터
    - KlassOop와 Class 객체 차이 (9장에서 자세히)
    
    ![image](https://github.com/mash-up-kr/S3A/assets/74983448/3222813b-d49c-4db8-90a7-a80a24029d36)
- 압축 oop 기법 제공
    - 대부분 기계어 워드라서, 메모리가 크게 낭비될 우려
    - 핫스팟이 조금이라도 메모리를 절약할 수 있도록
    - 힙에 있는 모든 객체의 Klass 워드, 참조형 인스턴스 필드, 객체 배열의 각 원소가 압축
- 핫스팟 객체 헤더
    - Mark 워드 (32비트 환경면 2바이트, 64비트면 8바이트)
    - Klass 워드 (압축될 수 있음)
    - 객체가 배열이면 length 워드 (항상 32비트)
    - 32비트 여백 (정렬 규칙 때문에 필요할 경우)
    
    ![image](https://github.com/mash-up-kr/S3A/assets/74983448/ed7fc8f8-16c4-42f0-a37c-8ae94c50a128)
    - 힙 크기 증가 (10~15%)로 압축을 안해도 되지만, 성능이 크게 개선되지는 않음
- 자바 객체도 배열
    - Klass 워드 다음에 배열 길이를 나타내는 Length 워드
- JVM 환경에서 자바 레퍼런스는 instanceOop 를 제외한 어떠한 것도 가리킬 수 없음
- 전체 상속구조
    
    ```java
    oop(추상 베이스)
    	instanceOop (인스턴스 객체)
    	methodOop (메서드 표현형)
    	arrayOop (배열 추상 베이스)
    	symbolOop (내부 심볼 / 스트링 클래스)
    	klassOop (Klass 헤더 / 자바 7 이전만 해당)
    	markOop
    ```
    

## 6.2.2 GC 루트 및 아레나

- GC 루트: 메모리의 고정점(anchor point)로, 메모리 풀 외부에서 내부를 가리키는 포인터
    - 내부 포인터: 메모리 풀 내부에서 같은 메모리 pool 내부의 다른 메모리를 가리킴
    - 외부 포인터: 내부 포인터와 정반대
    - GC 루트 종류: 스택 프레임, JNI, 레지스터, 코드 루트, 전역 객체, 로드된 클래스의 메타데이터
- 아레나: 핫스팟 GC가 작동하는 메모리 영역
- 핫스팟은 자바 힙을 관리할 때 시스템 콜을 하지 않음 → 핫스팟은 유저 공간 코드에서 힙 크기를 관리하므로 단순 측정값을 이용해 GC 서브시스템이 어떤 성능 문제를 일으키고 있는 파악 가능

## 6.3 할당과 수명

- 자바 애플리케이션에서 가비지 수집이 일어나는 주된 원인: 할당률, 객체 수명
- 할당률: 일정 기간 새로 생성된 객체가 사용한 메모리량, 값을 비교적 쉽게 측정 가능
- 객체 수명: 대부분 측정 어려움, 더 예민한 요인

### 6.3.1 약한 세대별 가설

- 이론적 근간: 시스템의 런타임 작용을 관찰하며 알게된 경험과 지식으로 JVM이 메모리를 관리
    - 거의 대부분의 객체는 아주 짧은 시간만 살아 있다
    - 나머지 객체는 기대 수명이 훨씬 길다
- 결론
    - 가비지를 수집하는 힙은, 단명 객체를 쉽고 빠르게 수집할 수 있게 설계해야 함
    - 장수 객체와 단명 객체를 완전히 떼어놓는 게 가장 좋음
- 핫스팟이 응용하는 메커니즘
    - 객체마다 ‘세대 카운트’를 센다
    - 큰 객체를 제외한 나머지 객체는 에덴 공간 생성. 살아남은 객체는 다른 곳으로 이동
    - 장수했다고 할 정도로 충분히 오래 살아남은 객체들은 별도의 메모리 영역에 보관
    
    ![image](https://github.com/mash-up-kr/S3A/assets/74983448/5fd3547f-5637-41f0-967d-655404725a4c)

- 외부에서 영 세대 내부를 가리키는 포인터를 계속 추적하는 기법 → GC 사이클에서 살아남은 젊은 객체 탐지를 위한 전체 그래프 순회 생략 가능

**Card table**

- 핫스팟에서 늙은 객체가 젊은 객체를 참조하는 정보를 기록하는 자료구조
- JVM이 관리하는 바이트 배열
- 핵짐 로직
    - 늙은 객체 o 에 있는 참조 필드가 변경되면 instanceOop 가 들어있는 카드를 찾아 더티 마킹
    - 레퍼런스 필드를 갱신할 때 마다 단순 쓰기 배리어를 이용
    - 필드 저장이 끝나면 다음을 실행
    
    ```java
    // 카드 테이블이 512바이트라서 9비트 우측 시프트 (2^9 = 512)
    
    cards[*instanceOop >> 9] = 0;
    ```
    
- 하지만 이제 G1(garbage first) 로 변경

## 6.4 핫스팟의 가비지 수집

- 메모리 관리
    - C/C++: OS를 이용해 동적으로 메모리 관리
    - JAVA: JVM이 메모리를 할당하고 유저 공간에서 연속된 단일 메모리 풀을 관리
- 객체는 보통 에덴 영역에 생성되며 GC가 계속 객체를 이동시킴(=방출)
- 핫스팟 수집기는 대부분 방출 수집기

### 6.4.1 스레드 로컬 할당

- 에덴에서 객체가 생성되고, 단명 객체는 다른 영역에 들어가지 못하고 에덴에서 수집
- 따라서 에덴은 가장 관리가 잘 되어야 하는 영역
- JVM은 에덴을 여러 버퍼로 나누어 애플리케이션 스레드가 새 객체를 할당하는 구역으로 활용
- 이러한 구역을 스레드 로컬 할당 버퍼 (thread-local allocation buffer, TLAB)
    - 배타적으로 제어하기 때문에 스레드 할당 복잡도 O(1)
    - 스레드가 객체 생성할 때 이 객체에 저장 공간이 할당되고, 스레드 로컬 포인터는 그다음 비어 있는 메모리 주소를 가리키도록 갱신

![image](https://github.com/mash-up-kr/S3A/assets/74983448/7ea0f031-9bf1-4e19-9e3e-2936fbfc524c)
### 6.4.2 반구형 수집

- 두 공간을 사용하는 특이한 수집기
- 장수하지 못할 객체를 임시 수용소에 담아 두는 아이디어
- 단명 객체가 테뉴어드(old) 영역을 어지럽히지 않게 하고 풀 GC 의 발생 빈도를 감소
- 기본적인 특성
    - 수집기는 live 반구를 수집할 때 객체를 다른 반구로 압착시켜 옮기고 수집된 반구는 비워서 재사용한다
    - 절반의 공간은 항상 완전히 비워둔다
- 실제로 공간을 2배로 사용하기 때문에 다소 비효율적 but 공간이 너무 크지 않다면 상당히 효율적
- 서바이버 공간: 핫스팟에서 영 힙의 반구부

## 6.5 병렬 수집기

- 자바 8 이전까지의 디폴트 가비지 수집기
- 종류
    - **Parallel GC:** 가장 단순한 young 세대용 병렬 수집기
    - **ParNew GC:** CMS 수집기와 함께 사용할 수 있도록 Parallel GC를 변형했다
    - **ParallelOld GC:** old(tenured) 수집기
- 공통 특징: 여러 스레드를 이요해 가급적 빠른 시간 내에 살아 있는 객체를 식별하고 기록 작업 최소화

### 6.5.1 영 세대 병렬 수집

- 가장 흔한 가비지 수집 형태
- 발생 상황: 스레드가 에덴에 객체를 할당하려는데 자신이 할당받은 TLAB 공간은 부족하고 JVM은 새 TLAB 할당할 수 없을 때
- 과정
    - 전페 애플리케이션 스레드 중단
    - 핫스팟은 영 세대를 뒤져서 가비지 아닌 객체를 골라냄 (GC루트를 출발점으로)
    - Parallel GC는 살아남은 객체를 현재 비어 있는 서바이버 공간으로 모두 방출
    - 세대 카운트를 늘려 한 차례 이동했음을 기록
    - 에덴과 이제 막 객체들을 방출시킨 서바이버 공간을 재사용 가능한 빈 공간으로 표시
    - 애플리케이션 스레드 재시작

![image](https://github.com/mash-up-kr/S3A/assets/74983448/a159f0e8-1bd9-49f5-bd62-79ba396fa95c)
![image](https://github.com/mash-up-kr/S3A/assets/74983448/252841c6-2fad-4674-9d11-943cdc00bcc2)
- 살아 있는 객체만 건드려 약한 세대별 가설의 이점 최대 활용

### 6.5.2 올드 세대 병렬 수집

- 자바 8부터의 디폴트 올드 세대 수집기
- Parallel GC와의 차이점: 하나의 연속된 메모리 공간에서 압착하는 수집기
- 더 이상 방출할 공간이 없으면 병렬 수집기는 올드 세대 내부에서 객체들을 재배치해서 늙은 객체가 죽고 빠져 버려진 공간을 최수
- 메모리 사용 면에서 효율적이고, 메모리 단편화 X
- 영 세대 수집은 단명 객체를 처리하기 때문에 변화가 많지만, 가끔 큰 객체가 올드 공간에 생성되는 것 말고는 수집이 일어나 재배치 하는 것 말고는 큰 변화 X

![image](https://github.com/mash-up-kr/S3A/assets/74983448/80817b4d-7122-4786-b40e-717ced3317ec)
### 6.5.3 병렬 수집기의 한계

- 병렬 수집기 목적: 전체를 대상으로 한번에, 가능한 효율적으로 GC를 수행함
- parallel gc 의 단점: 풀 GC가 일어난다는 점
- 영 수집
    - 극 소수의 객체만 살아남기 때문에 STW가 별 문제는 없지만 이것은 중단시간이 매우 짧다는 가정
- 올드 수집
    - 풀 디폴트 크기 자체가 영 세대의 7배
    - 영역 내 살아 있는 객체 수만큼 마킹 시간도 늘어남
    - 큰 약점: STW 시간이 힙 크기에 거의 비례

## 6.6 할당의 역할

- GC는 유입된 메모리 할당 요청을 수용하기에 메모리가 부족할때 필요한 만큼의 메모리를 공급
- 즉, GC 사이클은 어떤 고정된, 예측 가능한 일정에 맞춰 발생하는 게 아니라, 그때그때 필요로 발생
- TODO…

## 6.7 마치며

- 가비지 수집은 자바 커뮤니티에서 아주 활발하게 논의된 주제
- 핵심 개념들 꼭 기억하기 ~~

## 참고

- 이미지: https://blog.yevgnenll.me/posts/basic-of-garbage-collector
