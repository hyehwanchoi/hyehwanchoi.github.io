---
layout: single
title:  "실시간 위치 데이터 처리 시스템 개선기 (2): 120초 Timeout 로그에서 병목을 찾기까지"
date:   2026-08-11 09:30:00 +0900
lastmod : 2026-08-11 15:00:00 +0900
sitemap :
changefreq : daily
priority : 1.0
author: HyeHwan Choi
categories: project
tags:   kafka vertx zookeeper backend realtime troubleshooting performance
---

[지난 글](/project/realtime-location-data-project/)에서는 일정 시간 데이터를 모아 처리하던 구조를 실시간 흐름으로 바꾸면서 어떤 원칙을 세웠는지 정리했습니다.

실시간 처리로 전환한 뒤에는 한숨 돌릴 수 있을 거라고 생각했습니다. 그런데 처리 주기가 짧아지자, 그동안 숨어 있던 지연과 실패가 기다렸다는 듯 로그에 함께 나타나기 시작했습니다.

- Producer record가 120초 후 만료됐다.
- Vert.x 이벤트 루프가 3~6초 동안 멈췄다.
- Consumer가 offset을 commit하는 순간 group rebalance 오류가 발생했다.
- Kafka broker의 `Node disconnected` 로그가 반복됐다.
- ZooKeeper는 일부 client의 session 요청을 거부했다.

솔직히 처음에는 Kafka 클러스터가 불안정해진 것으로 봤습니다. timeout, disconnect, rebalance가 비슷한 시간에 찍혔기 때문입니다. 하지만 로그를 하나씩 코드 흐름 위에 올려놓자 서로 다른 계층의 문제가 한꺼번에 보였을 뿐이라는 사실이 드러났습니다.

이번 글에서는 각각의 로그가 어느 계층에서 발생했고, 어떤 코드와 설정을 함께 확인했는지 정리합니다. 회사명, 서버 주소, 실제 topic 이름, 장비 식별자는 일반화했습니다. 아직 측정하지 않은 성능 수치나 적용하지 않은 개선은 완료된 결과처럼 적지 않았습니다.

---

## 1. 장애를 데이터 흐름 위에 놓고 보기

먼저 로그를 개별적으로 보지 않고 전체 처리 흐름 위에 배치했습니다.

```text
수집 장비
  -> CMD: 패킷 파싱 및 topic 분배
  -> Kafka
  -> PMD: MAC별 버퍼링 및 위치 계산
  -> Kafka
  -> 후속 시스템
```

이렇게 보면 같은 `timeout`이라도 발생 지점과 의미가 다르다는 것을 알 수 있습니다.

| 증상 | 발생 계층 | 먼저 확인할 대상 |
|---|---|---|
| Producer record 만료 | Kafka Producer | broker 연결, metadata, partition, buffer, ack |
| Vert.x blocked thread | 애플리케이션 | 이벤트 루프에서 수행한 동기 작업 |
| Consumer commit 실패 | Kafka Consumer | poll 주기, 처리시간, rebalance |
| Node disconnected | Kafka client/broker | idle 연결인지 실제 broker 장애인지 |
| zxid가 뒤처진 서버의 session 거부 | ZooKeeper | quorum 상태, dataDir, peer 동기화 |

문제의 범위를 나눈 뒤에야 설정을 무작정 바꾸지 않고 확인 순서를 세울 수 있었습니다.

---

## 2. Producer Timeout은 10건 버퍼의 시간초과가 아니었다

PMD에서 다음과 같은 형태의 오류가 발생했습니다.

```text
failed to publish buffer. reason=size, count=10

TimeoutException:
Expiring N record(s) for topic-partition:
120000 ms has passed since batch creation
```

저도 처음에는 `reason=size`, `count=10`, `120000 ms`를 한 덩어리로 읽었습니다. 그래서 다음처럼 이해했습니다.

> 10건을 모으려고 기다리다가 애플리케이션 버퍼가 120초 후 만료됐다.

코드를 따라가 보니 예상이 틀렸습니다. 실제로는 서로 다른 두 단계의 로그였습니다.

`reason=size`는 PMD가 특정 MAC의 데이터를 10건 모아 위치 계산과 후속 publish를 시작했다는 의미입니다. 반면 `120000 ms has passed since batch creation`은 그 이후 Kafka Producer에 전달된 record가 제한시간 안에 broker로 전송되지 못했다는 의미입니다.

| 구분 | 목적 | 완료 조건 | 실패가 의미하는 것 |
|---|---|---|---|
| MAC별 애플리케이션 버퍼 | 위치 계산에 사용할 스캔 집계 | 10건 또는 제한시간 | 위치 계산을 언제 시작할 것인가 |
| Kafka Producer 배치 | 여러 record의 전송 효율 향상 | `batch.size`, `linger.ms`, broker 응답 | record가 Kafka에 저장됐는가 |

Kafka Producer의 `batch.size`와 `linger.ms`는 애플리케이션이 10건을 모으는 시간과 직접적인 관계가 없습니다. 또한 Consumer lag가 Producer timeout의 직접 원인이라고 단정할 수도 없습니다. 같은 broker의 자원 부족이나 전체 시스템 부하가 두 현상을 동시에 만들 수는 있지만, 각각의 지표를 따로 확인해야 합니다.

Producer timeout에서는 다음 순서로 범위를 좁힙니다.

1. 대상 broker와 연결할 수 있었는가
2. topic과 partition의 metadata를 정상적으로 얻었는가
3. 특정 partition에 key가 몰리지 않았는가
4. broker가 요청을 처리하고 ack를 반환했는가
5. Producer buffer가 고갈되거나 record가 `delivery.timeout.ms`를 넘지 않았는가

핵심은 `send()`를 호출한 것과 Kafka에 저장이 완료된 것을 구분하는 것입니다.

---

## 3. 10건이 안 들어오는 장비도 처리해야 했다

위치 계산에는 여러 AP에서 수집한 신호가 필요합니다. 그래서 PMD는 MAC별로 데이터를 모으고 기본적으로 10건이 되면 계산을 시작합니다.

```text
데이터 수 >= 10
    -> reason=size로 즉시 처리

데이터 수 < 10이고 제한시간 경과
    -> reason=timeout으로 처리
```

크기 조건만 사용하면 유입량이 적은 장비의 데이터는 계속 버퍼에 남습니다. 반대로 시간 조건만 사용하면 트래픽이 충분한 상황에서도 불필요하게 계산을 늦추게 됩니다. 그래서 크기와 시간을 함께 사용하는 `size-or-time` 방식이 필요했습니다.

현재 처리 흐름에는 다음 조건도 포함되어 있습니다.

- 잘못된 MAC과 허용하지 않는 MAC은 AP 조회와 버퍼 생성 전에 제외한다.
- 알 수 없는 AP 데이터는 위치 계산 대상에 넣지 않는다.
- 주기적으로 오래된 MAC 버퍼를 찾아 `reason=timeout`으로 처리한다.
- partition이 revoke되거나 프로세스가 종료될 때 남은 버퍼를 flush한다.

다만 여기서 더 중요한 문제가 있었습니다. 현재 구현은 MAC 버퍼를 먼저 Map에서 제거한 뒤 Kafka Producer의 비동기 `send()`를 호출합니다.

```text
버퍼 제거
  -> 위치 계산
  -> Kafka send
  -> 비동기 성공 또는 실패
```

이 상태에서 publish가 실패하면 메모리 버퍼는 이미 제거된 상태입니다. Consumer가 auto commit을 사용한다면 입력 offset과 출력 publish의 성공 여부도 하나의 처리 단위로 묶이지 않습니다.

따라서 다음 정책을 코드로 명확히 해야 합니다.

- publish 성공 후에만 입력 처리를 완료한 것으로 볼 것인가
- 실패한 버퍼를 메모리에 되돌릴 것인가, 별도 retry queue나 DLQ로 보낼 것인가
- 재시도 중복을 어떤 key로 식별할 것인가
- 종료와 rebalance 시 비동기 publish 완료를 어디까지 기다릴 것인가

이 부분은 아직 검증이 필요한 개선 항목입니다. 단순히 timeout 값을 늘리는 것보다 데이터 유실과 중복의 경계를 먼저 정의해야 합니다.

---

## 4. Vert.x 이벤트 루프가 멈춘 이유

CMD에서는 다음과 같은 경고가 반복됐습니다.

```text
Thread vert.x-eventloop-thread-N has been blocked
for 3803 ms, time limit is 2000 ms
```

이 로그는 Kafka broker가 느리다는 의미가 아니라 Vert.x 이벤트 루프의 작업이 제한시간 안에 반환되지 않았다는 의미입니다.

CMD의 Wi-Fi Consumer 흐름을 확인해 보니 일반 Verticle의 `setPeriodic` 핸들러에서 다음 작업을 순서대로 수행하고 있었습니다.

```text
Event Loop
  -> 동기 KafkaConsumer.poll()
  -> 패킷 파싱
  -> DB session 생성 및 조회
  -> 여러 배치 목록 생성
  -> Event Bus 전송
  -> KafkaConsumer.commitSync()
```

스택 트레이스를 처음 봤을 때는 `poll()` 하나가 문제처럼 보였습니다. 하지만 코드를 따라가니 최대 500ms를 기다리는 `poll()` 뒤로 DB 조회, 대량 반복 처리, 동기 commit이 같은 이벤트 루프에서 이어지고 있었습니다. 하나씩 보면 짧아 보이는 호출도 한 handler 안에 쌓이면 2초 제한을 쉽게 넘겼습니다.

Vert.x 이벤트 루프는 짧고 non-blocking인 작업에 적합합니다. `blockedThreadCheckInterval`을 늘리면 경고가 늦게 출력될 뿐, 다른 이벤트 처리가 지연되는 문제는 그대로 남습니다.

개선 방향은 작업의 책임과 실행 스레드를 분리하는 것입니다.

```text
Kafka Consumer Worker
  -> 패킷 파싱 Use Case
  -> DB Worker 또는 Event Bus
  -> 처리 결과 확인
  -> Offset Commit
```

여기서 패킷 파싱, 상태 판정, T_ID 필터, topic 결정은 Kafka나 DB를 모르는 순수 로직으로 분리할 수 있습니다. 그래야 입력 하나에 어떤 결과가 나오는지 빠르게 단위 테스트할 수 있고, 느린 I/O는 worker 영역에서만 실행할 수 있습니다.

---

## 5. Consumer rebalance와 commit 실패

blocked-thread 경고와 함께 다음 오류도 확인했습니다.

```text
Commit cannot be completed since the group has already rebalanced
and assigned the partitions to another member
```

Kafka Consumer는 정해진 주기 안에 계속 `poll()`을 호출해야 합니다. 메시지 처리에 너무 오래 걸리면 Consumer Group은 해당 consumer가 정상적으로 동작하지 않는다고 판단하고 partition을 다른 member에게 재할당할 수 있습니다.

이미 partition 소유권을 잃은 consumer가 `commitSync()`를 호출하면 commit이 거부됩니다.

다만 3~6초의 blocked-thread 로그 하나만으로 `max.poll.interval.ms` 초과를 증명할 수는 없습니다. 반복되는 처리 지연, 실제 poll 간격, rebalance 시각을 함께 확인해야 합니다. 구조적으로 같은 handler 안에 blocking 작업이 모여 있다는 사실은 강한 의심 지점이지만 로그의 시간 상관관계까지 확인해야 원인이라고 결론 내릴 수 있습니다.

확인하고 조정할 항목은 다음과 같습니다.

- 실제 `poll()` 호출 간격과 batch 처리시간을 기록한다.
- `max.poll.records`를 한 번에 처리 가능한 크기로 줄인다.
- 긴 후속 처리가 필요하면 consumer를 pause하고 worker에 전달한다.
- 처리 성공을 확인한 뒤 offset을 commit한다.
- `max.poll.interval.ms`는 측정한 최대 처리시간을 기준으로 잡는다.
- Consumer 수와 partition 수가 실제 병렬 처리 구조와 맞는지 확인한다.

`max.poll.interval.ms`를 먼저 늘리는 것은 장애 발생 시점을 늦출 수는 있지만 처리 병목을 없애지는 못합니다.

---

## 6. Node disconnected와 ZooKeeper 로그는 따로 봐야 했다

Kafka client에는 다음과 같은 INFO 로그가 출력될 수 있습니다.

```text
Node N disconnected
```

이 로그 하나만으로 broker 장애라고 판단할 수는 없습니다. idle connection 정리일 수도 있고, broker 재시작이나 네트워크 단절일 수도 있습니다. 같은 시간의 broker 상태, reconnect 성공 여부, request timeout, leader 변경을 함께 봐야 합니다.

ZooKeeper에서는 다음 로그가 반복됐습니다.

```text
Refusing session request for client ...
as it has seen zxid 0x...
our last zxid is 0x...
client must try another server
```

client가 기억하는 zxid보다 접속한 ZooKeeper 서버의 최신 zxid가 과거이기 때문에 해당 서버가 session 요청을 받아들이지 않은 상황입니다. 이때는 client 재시도만 볼 것이 아니라 ensemble의 각 서버가 같은 quorum에 합류했는지, dataDir과 `myid`가 올바른지 확인해야 합니다.

ZooKeeper 구성에서 다시 확인한 내용은 다음과 같습니다.

- `2888` 계열 포트는 follower와 leader의 데이터 동기화에 사용한다.
- `3888` 계열 포트는 leader election에 사용한다.
- 각 인스턴스의 client port, dataDir, `myid`, peer port가 충돌하면 안 된다.
- 같은 물리 서버에 인스턴스를 두 개 띄워 홀수를 맞춰도 그 서버가 내려가면 두 표를 동시에 잃는다.
- 홀수 개수 자체보다 서로 다른 장애 도메인에 배치하는 것이 중요하다.
- 개발 환경에서 ensemble 데이터를 초기화할 때는 모든 프로세스가 완전히 종료됐는지 PID와 listening port로 먼저 확인해야 한다.

정상적으로 다시 합류하면 `UPTODATE`, `FOLLOWING`, `broadcast`와 같은 상태 전환을 확인할 수 있습니다.

---

## 7. 성능 개선은 설정 하나로 끝나지 않는다

이번 로그를 정리하면서 개선 대상을 세 영역으로 나눴습니다.

### CMD

- 이벤트 루프에서 동기 Kafka 및 DB 작업을 분리한다.
- polling, 패킷 파싱, DB 저장의 책임을 분리한다.
- T_ID 필터와 topic 분배를 순수 로직으로 추출한다.
- DB 저장 성공과 offset commit의 관계를 명확히 한다.
- downstream이 느릴 때 무제한으로 Event Bus에 쌓이지 않도록 backpressure를 둔다.

### PMD

- MAC별 size-or-time 버퍼링을 유지한다.
- 잘못된 MAC과 AP를 가능한 앞 단계에서 제외한다.
- AP와 tracker 정보는 주기적으로 갱신되는 snapshot cache로 조회한다.
- publish 실패 시 retry, DLQ, 중복 방지 정책을 정의한다.
- 종료와 rebalance 시 잔여 버퍼의 publish 완료를 추적한다.

### Kafka

- topic 수가 아니라 partition 수와 consumer 수를 함께 본다.
- MAC 기반 key가 특정 partition에 집중되는지 확인한다.
- partition별 유입량과 Consumer lag 편차를 확인한다.
- Producer timeout이 발생한 partition과 broker 상태를 연결해서 본다.
- 설정 변경 전후에는 동일한 부하를 사용한다.

이 중 size-or-time 버퍼, MAC 필터, AP·tracker cache처럼 현재 적용된 항목과 publish 재시도 및 commit 경계처럼 추가 구현이 필요한 항목을 구분해서 관리하고 있습니다.

---

## 8. 무엇을 측정해야 개선이라고 말할 수 있을까

테스트에서는 1분에 약 140만 건 수준의 최대 유입량을 고려했습니다. 하지만 단순히 요청이 성공했는지만으로는 실시간 처리 성능을 판단하기 어렵습니다.

JMeter와 Kafka UI, 애플리케이션 로그를 함께 사용해 다음 항목을 같은 시간축에서 봐야 합니다.

| 지표 | 확인하려는 내용 |
|---|---|
| 초당 처리량 | 목표 유입량을 지속해서 처리하는가 |
| p95·p99 처리 지연 | 일부 메시지만 오래 지연되지는 않는가 |
| partition별 lag | 특정 partition에 부하가 집중되는가 |
| lag 정상화 시간 | burst 이후 실시간 상태로 돌아오는가 |
| Producer timeout | 전송하지 못한 record가 있는가 |
| blocked-thread 경고 | 이벤트 루프가 I/O에 막히는가 |
| rebalance 횟수 | 처리 지연으로 group이 불안정해지는가 |
| 유실·중복 건수 | retry와 commit 정책이 의도대로 동작하는가 |

테스트 시나리오도 정상 부하만으로 끝내지 않습니다.

1. 평상시 유입량을 장시간 유지한다.
2. 짧은 시간 동안 유입량을 급격히 높인다.
3. DB 응답을 지연시킨다.
4. Kafka broker 하나를 일시 중단한다.
5. 처리 중 consumer를 재시작해 rebalance를 발생시킨다.
6. 특정 MAC에는 10건 미만의 데이터만 전달한다.
7. 특정 key를 집중시켜 partition skew를 만든다.

현재 공개할 수 있는 반복 측정 결과가 충분하지 않은 항목은 숫자를 임의로 채우지 않았습니다. 동일 조건에서 최소 3회 측정한 뒤 중앙값과 p95·p99를 비교해야 개선 효과라고 말할 수 있습니다.

---

## 9. 재발을 막기 위한 테스트

현재 MAC 필터에는 다음 단위 테스트가 적용돼 있습니다.

- 허용한 세 번째 MAC octet만 통과하는가
- 대소문자를 구분하지 않는가
- null, 빈 문자열, 잘못된 길이와 16진수를 거부하는가
- 쉼표로 구분한 여러 prefix를 처리하는가
- 설정이 없거나 잘못되면 시작 시 실패하는가

하지만 전체 전달 흐름을 보호하려면 테스트 범위를 더 넓혀야 합니다.

- T_ID 버킷에 따라 포함·제외가 정확한지
- MAC의 마지막 값에 따라 같은 topic으로 안정적으로 분배되는지
- 10건 도달 시 `reason=size`로 한 번만 flush되는지
- 1~9건이 제한시간 후 `reason=timeout`으로 flush되는지
- publish 실패 후 버퍼가 유실되지 않는지
- 처리 성공 전에 offset이 commit되지 않는지
- rebalance와 종료 시 잔여 데이터가 처리되는지
- 동일한 입력에 리팩터링 전후 payload가 같은지
- DB가 느려도 Vert.x 이벤트 루프가 차단되지 않는지

특히 시간 기반 버퍼는 실제로 몇 분을 기다리는 테스트보다 `Clock`과 scheduler를 주입해 가상 시간으로 검증할 수 있어야 합니다. Kafka, DB, Vert.x와 분리된 use case가 필요한 이유이기도 합니다.

---

## 마무리

처음에는 설정 하나만 찾으면 끝날 거라고 기대했습니다. `delivery.timeout.ms`나 `max.poll.interval.ms` 같은 값을 조정하면 조용해질 것 같았습니다. 하지만 로그를 따라갈수록 문제는 한 군데에 있지 않았습니다.

Producer timeout은 전송 계층에서 발생했고, rebalance는 Consumer의 poll과 처리시간을 확인해야 했으며, blocked-thread는 애플리케이션의 스레드 사용 방식에서 시작됐습니다. ZooKeeper의 zxid 오류는 다시 별도의 quorum과 데이터 동기화 문제였습니다.

이번 기록은 모든 문제가 해결됐다는 성공담은 아닙니다. 오히려 publish 실패 뒤 사라질 수 있는 버퍼처럼 아직 위험한 경계도 남아 있습니다. 다만 비슷한 시각의 로그를 하나의 원인으로 성급히 묶지 않고, 전체 데이터 흐름 위에서 코드와 지표를 연결해 보는 습관은 얻었습니다.

다음 글에서는 이 과정에서 드러난 전역 설정, Kafka·DB 직접 의존성, 큰 Verticle을 어떻게 테스트 가능한 구조로 분리할 수 있는지 정리해보려고 합니다.

> 실시간 위치 데이터 처리 시스템 개선기 (3): 레거시 Kafka·Vert.x 코드를 테스트 가능한 구조로 바꾸기
