# How to solve concurrency issues

## 일반적인 재고 감소 로직

여러 스레드에서 메소드에 접근을 시도하는 경우, 레이스 컨디션 발생

> **레이스 컨디션** : 둘 이상의 스레드가 공유 데이터에 엑세스 할 수 있고 동시에 변경을 시도할 때 생기는 문제

## Java의 `synchronized`를 이용한 문제 해결

하나의 프로세스 내에서는 접근을 제어할 수 있음.

=> 서버가 여러 개인 경우 레이스 컨디션 발생


## MySQL를 활용한 문제 해결

1. **Pessimistic Lock**

    - 실제로 **데이터에 Lock을 걸어** 데이터의 정합성을 맞추는 방법
    - exclusive lock을 걸게되면 다른 트랜잭션에서는 lock이 해제되기 전에 데이터를 가져갈 수 없음
    - **데드락** 발생 가능성 有

2. **Optimistic Lock**

    - 실제로 Lock을 이용하지 않고 **버전**을 이용하여 정합성을 맞추는 방법
    - 먼저 데이터를 읽은 후 update를 수행할 때 현재 내가 읽은 버전이 맞는지 확인하여 업데이트
    - 내가 읽은 버전에서 수정사항이 생겼을 경우에는 application에서 다시 읽은 후에 작업 수행

3. **Named Lock**

    - 이름을 가진 metadata locking
    - 이름을 가진 lock을 획득한 후 해제할 때까지 다른 세션은 이 lock을 획득할 수 없도록 함
    - transaction이 종료될 떄까지 **lock이 자동으로 해제되지 않음**
        1. 별도의 **명령어**로 해제
        2. **선점 시간**이 끝나야 해제

### Pessimistic lock vs Named lock

- Pessimistic lock
  - row나 table 단위로 lock
  - timeout을 구현하기 어려움

- Named lock
  - metadata에 lock
  - 분산락 구현 시 사용
  - timeout 구현이 쉬움

### Pessimistic lock vs Optimistic Lock

- Pessimistic lock
  - 충돌이 빈번한 경우 Optimistic Lock보다 성능이 더 좋음
  - Lock으로 업데이트 제어 => 데이터 정합성 보장
  - 별도의 lock을 잡기 때문에 성능 감소
  
- Optimistic Lock
  - 별도의 lock을 잡지 않아 Pessimistic lock보다 성능상 이점이 있음
  - 업데이트 실패 시 재시도 로직을 작성해야함

> 충돌이 빈번하다면 **Pessimistic lock**, 충돌이 빈번하지 않다면 **Optimistic Lock**


## Redis를 활용한 문제 해결

### Lettuce

- `setnx` 명령어를 활용하여 분산락 구현
  - `setnx` (set if not exist) : 기존 값이 없는 경우에만 key/value set
- spin lock 방식
  - **spin lock** : lock을 획득하려는 스레드가 lock을 사용할 수 있는지 확인하면서 반복적으로 lock 획득 시도
  - retry 로직 작성 필요
- MySQL의 Named Lock과 유사하나 세션 관리를 신경쓰지 않아도 됨

### Redisson

- pub/sub 기반으로 Lock 구현 제공
  - 채널을 하나 생성하고 락을 점유 중인 스레드가 락을 획득하기 위해 대기 중인 스레드에게 해제를 알려주면 안내를 받은 스레드가 락 획득 시도
  - 별도의 retry 로직 작성 필요 X

### Lettuce vs Redisson

- Lettuce
  - 구현이 간단함
  - spring-data-redis 이용 시 lettuce가 기본이므로 별도의 라이브러리를 사용하지 않아도 됨
  - **spin lock** 방식 => 동시에 많은 스레드가 lock 획득 대기 상태라면 **redis에 부하**가 갈 수 있음

- Redisson
  - 락 획득 재시도를 기본으로 제공함
  - **pub-sub** 방식으로 구현 => lettuce에 비해 redis에 **부하가 적음**
  - [별도의 라이브러리](https://mvnrepository.com/artifact/org.redisson/redisson-spring-boot-starter) 사용
  - lock을 라이브러리 차원에서 제공해주므로 사용법을 공부해야함

> 재시도가 필요하지 않은 lock은 **lettuce**, 재시도가 필요한 lock은 **redisson**


## MySQL vs Redis

- MySQL
  - 이미 MySQL을 사용 중이라면 별도의 비용없이 사용 가능
  - 어느 정도의 트래픽까지는 문제없이 활용 가능
  - Redis에 비해 성능이 좋지 않음

- Redis
  - 활용 중인 Redis가 없다면 별도의 구축 비용과 인프라 관리 비용 발생
  - MySQL보다 성능이 좋음

---

reference

- [인프런 - 재고시스템으로 알아보는 동시성이슈 해결방법](https://inf.run/FrsT)
- [레디스를 활용한 분산 락(Distrubuted Lock) feat lettuce, redisson](https://0soo.tistory.com/256)
- [MySQL을 이용한 분산락으로 여러 서버에 걸친 동시성 관리](https://techblog.woowahan.com/2631/)
- [풀필먼트 입고 서비스팀에서 분산락을 사용하는 방법 - Spring Redisson](https://helloworld.kurly.com/blog/distributed-redisson-lock/)
- [Weekly Java: 간단한 재고 시스템으로 학습하는 동시성 이슈](https://sigridjin.medium.com/weekly-java-간단한-재고-시스템으로-학습하는-동시성-이슈-9daa85155f66)
- [Redisson Github](https://github.com/redisson/redisson)