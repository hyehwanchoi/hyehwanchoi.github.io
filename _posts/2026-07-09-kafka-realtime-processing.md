---
layout: single
title:  "Kafka 기본 개념 정리"
date:   2026-07-09 12:00:00
lastmod : 2026-07-09 13:10:00
sitemap :
changefreq : daily
priority : 1.0
author: HyeHwan Choi
categories: study
tags:   kafka backend realtime
---

최근 대용량 실시간 데이터를 처리하는 프로젝트를 진행하면서 Kafka를 다시 깊게 보게 되었다.

Kafka는 단순히 메시지를 주고받는 도구처럼 보이지만, 실제로 운영 환경에서 사용해보면 topic, partition, offset, consumer group 같은 개념을 제대로 이해하는 것이 중요하다는 것을 느끼게 된다.

이번 글에서는 Kafka를 사용할 때 꼭 알아야 하는 기본 개념들을 정리해보려고 한다.

---

## Kafka란?

Kafka는 대용량 이벤트 데이터를 안정적으로 저장하고 전달하기 위한 분산 메시징 플랫폼이다.

일반적인 시스템에서는 하나의 서비스가 다른 서비스에 직접 요청을 보내는 방식으로 데이터를 전달한다. 하지만 서비스가 많아지고 데이터 양이 커지면 직접 연결 방식은 점점 복잡해진다.

Kafka는 이 사이에 위치해서 데이터를 보내는 쪽과 받는 쪽을 분리해준다.

```text
Producer -> Kafka -> Consumer
```

데이터를 보내는 서비스는 Kafka에 메시지를 넣고, 데이터를 사용하는 서비스는 Kafka에서 메시지를 읽는다.

이 구조의 장점은 명확하다.

- producer와 consumer가 서로 직접 알 필요가 없다.
- consumer가 잠시 느려져도 Kafka에 메시지가 보관된다.
- 하나의 메시지를 여러 consumer group이 각자 읽을 수 있다.
- partition을 통해 병렬 처리가 가능하다.

Kafka는 단순한 큐라기보다 이벤트를 저장하고 여러 시스템에 전달하는 로그 기반 플랫폼에 가깝다.

---

## Topic

topic은 Kafka에서 메시지를 구분하는 논리적인 단위다.

예를 들어 위치 데이터를 처리하는 시스템이라면 다음처럼 topic을 나눌 수 있다.

```text
AOA-AOA-00
AOA-BLE-00
BLE-00
```

producer는 특정 topic으로 메시지를 보내고, consumer는 필요한 topic을 구독해서 메시지를 읽는다.

topic은 파일 시스템의 폴더처럼 생각할 수 있다. 같은 종류의 메시지를 모아두는 이름 공간이다.

다만 topic을 많이 만든다고 무조건 성능이 좋아지는 것은 아니다. Kafka에서 실제 병렬 처리와 밀접하게 관련된 것은 topic보다 partition이다.

---

## Partition

partition은 topic을 물리적으로 나눈 단위다.

하나의 topic은 하나 이상의 partition을 가질 수 있다.

```text
Topic: BLE-00
  Partition 0
  Partition 1
  Partition 2
  Partition 3
```

Kafka가 대용량 메시지를 처리할 수 있는 이유 중 하나가 바로 partition이다.

partition이 여러 개 있으면 메시지를 여러 partition에 나누어 저장할 수 있고, consumer도 partition 단위로 병렬 처리할 수 있다.

예를 들어 topic의 partition이 1개라면 consumer가 아무리 많아도 해당 topic은 사실상 한 줄로 처리된다.

반대로 partition이 8개이고 consumer도 충분히 있다면 여러 consumer가 나누어 처리할 수 있다.

```text
Partition 0 -> Consumer A
Partition 1 -> Consumer B
Partition 2 -> Consumer C
Partition 3 -> Consumer D
```

그래서 대용량 처리에서는 partition 수가 중요하다.

하지만 partition은 많을수록 무조건 좋은 것도 아니다.

- broker의 파일과 메타데이터가 늘어난다.
- consumer rebalance 비용이 커질 수 있다.
- key 기반 순서 보장 범위가 달라진다.
- 운영 중 partition 수를 줄일 수 없다.

Kafka에서 partition은 늘릴 수는 있지만 줄일 수 없다. 그래서 처음부터 너무 작게 잡으면 병목이 되고, 너무 크게 잡으면 운영 부담이 된다.

---

## Message Key

producer가 메시지를 보낼 때 key를 지정할 수 있다.

Kafka는 key를 기준으로 어떤 partition에 메시지를 넣을지 결정한다.

같은 key를 가진 메시지는 보통 같은 partition으로 들어간다.

```text
key = MAC_ADDRESS_A -> Partition 1
key = MAC_ADDRESS_B -> Partition 3
key = MAC_ADDRESS_A -> Partition 1
```

이게 중요한 이유는 순서 보장 때문이다.

Kafka는 topic 전체의 순서를 보장하지 않는다. Kafka가 보장하는 것은 partition 안에서의 순서다.

따라서 특정 장비, 사용자, 주문처럼 같은 기준의 메시지를 순서대로 처리해야 한다면 key 설계를 신중하게 해야 한다.

예를 들어 장비 위치 데이터라면 장비의 MAC 주소를 key로 사용할 수 있다. 그러면 같은 장비의 메시지는 같은 partition에 들어가 순서를 유지하기 쉽다.

하지만 특정 key에 메시지가 너무 몰리면 하나의 partition만 바빠질 수 있다. 순서 보장과 부하 분산 사이에서 균형을 잡아야 한다.

---

## Broker

broker는 Kafka 서버 한 대를 의미한다.

Kafka cluster는 여러 broker로 구성될 수 있다.

```text
Kafka Cluster
  Broker 0
  Broker 1
  Broker 2
```

각 broker는 topic의 partition 일부를 가지고 있다.

broker가 여러 대 있으면 partition을 여러 서버에 분산할 수 있고, replication을 통해 장애에도 대비할 수 있다.

단일 broker 환경에서는 구조가 단순하지만, broker 하나가 장애나면 Kafka 전체가 영향을 받을 수 있다. 운영 환경에서는 보통 여러 broker를 구성하고 replication factor를 설정한다.

---

## Replication

replication은 partition 데이터를 여러 broker에 복제하는 기능이다.

예를 들어 replication factor가 3이면 하나의 partition 데이터가 3개의 broker에 복제된다.

```text
Partition 0
  Leader: Broker 0
  Follower: Broker 1
  Follower: Broker 2
```

producer와 consumer는 보통 leader partition과 통신한다. follower는 leader의 데이터를 복제하다가, leader가 죽으면 새로운 leader가 될 수 있다.

replication이 있으면 broker 장애에 더 안전해진다.

하지만 replication factor를 높이면 저장 공간과 네트워크 비용도 증가한다. 안정성과 비용 사이의 선택이 필요하다.

---

## Producer

producer는 Kafka로 메시지를 보내는 클라이언트다.

producer는 메시지를 바로 broker로 하나씩 보내는 것이 아니라, 내부적으로 batch를 만들어 전송한다.

관련 설정으로는 다음과 같은 것들이 있다.

- `acks`
- `batch.size`
- `linger.ms`
- `buffer.memory`
- `retries`
- `delivery.timeout.ms`

예를 들어 `linger.ms`는 메시지를 보내기 전에 잠깐 기다리며 batch를 더 모을 수 있게 한다. 처리량을 높이는 데 도움이 될 수 있지만, 지연 시간은 늘어날 수 있다.

`acks`는 broker가 메시지를 받았다고 언제 producer에게 응답할지를 정한다.

```text
acks=0   응답을 기다리지 않음
acks=1   leader가 받으면 성공
acks=all replica까지 확인
```

안정성이 중요하면 `acks=all`이 좋지만, 처리 속도와 지연 시간 측면에서는 비용이 있다.

Kafka producer 설정은 정답이 하나가 아니라, 시스템이 중요하게 보는 기준에 따라 달라진다.

---

## Consumer

consumer는 Kafka에서 메시지를 읽는 클라이언트다.

consumer는 topic을 구독하고, 할당받은 partition에서 메시지를 읽는다.

Kafka에서 consumer를 이해할 때 가장 중요한 개념은 consumer group이다.

---

## Consumer Group

consumer group은 여러 consumer를 하나의 그룹으로 묶은 것이다.

같은 consumer group에 속한 consumer들은 topic의 partition을 나누어 처리한다.

```text
Topic: BLE-00, Partition 0~3

Consumer Group: pmd-group
  Consumer A -> Partition 0
  Consumer B -> Partition 1
  Consumer C -> Partition 2
  Consumer D -> Partition 3
```

같은 group 안에서는 하나의 partition을 동시에 여러 consumer가 읽지 않는다.

반대로 서로 다른 consumer group은 같은 메시지를 각자 읽을 수 있다.

```text
Consumer Group A -> 메시지 전체 읽음
Consumer Group B -> 메시지 전체 읽음
```

이 구조 덕분에 Kafka에서는 하나의 데이터를 여러 목적의 시스템이 독립적으로 사용할 수 있다.

예를 들어 하나의 위치 데이터 topic을 두고, 한 consumer group은 실시간 위치 계산을 하고, 다른 consumer group은 로그 저장이나 분석을 할 수 있다.

---

## Offset

offset은 partition 안에서 메시지의 위치를 나타내는 번호다.

Kafka는 각 partition의 메시지에 순서대로 offset을 부여한다.

```text
Partition 0
  offset 0
  offset 1
  offset 2
  offset 3
```

consumer는 자신이 어디까지 읽었는지를 offset으로 관리한다.

그래서 consumer가 중간에 죽었다가 다시 떠도, 마지막으로 commit한 offset부터 이어서 읽을 수 있다.

여기서 중요한 것이 offset commit이다.

consumer가 메시지를 읽었다고 해서 Kafka가 자동으로 "처리 완료"를 아는 것은 아니다. consumer가 offset을 commit해야 Kafka가 어디까지 처리했는지 알 수 있다.

자동 commit을 사용할 수도 있고, 직접 commit을 제어할 수도 있다.

실시간 시스템에서는 offset commit 시점이 중요하다.

- 메시지를 처리하기 전에 commit하면 장애 시 메시지를 잃을 수 있다.
- 메시지를 처리한 뒤 commit하면 장애 시 중복 처리될 수 있다.

대부분의 시스템은 중복 처리를 감수하더라도 메시지 유실을 피하는 쪽을 선택한다.

---

## auto.offset.reset

`auto.offset.reset`은 consumer group에 저장된 offset이 없을 때 어디서부터 읽을지를 정하는 설정이다.

대표적으로 두 값이 있다.

```text
earliest: 가장 오래된 메시지부터 읽음
latest: 새로 들어오는 메시지부터 읽음
```

주의할 점은 이 설정이 항상 적용되는 것이 아니라는 점이다.

이미 consumer group에 offset이 저장되어 있다면 `earliest`나 `latest`보다 저장된 offset이 우선이다.

즉 `latest`로 설정했다고 해서 서비스 재기동 시 항상 최신 메시지부터 읽는 것은 아니다.

정말 시작 시점에 최신 위치로 이동해야 한다면 consumer가 partition을 할당받은 뒤 명시적으로 `seekToEnd`를 해야 한다.

```java
consumer.partitionsAssignedHandler(partitions -> {
    for (TopicPartition partition : partitions) {
        consumer.seekToEnd(partition);
    }
});
```

하지만 이 방식은 기존에 쌓인 메시지를 건너뛰는 것이므로, 시스템 요구사항을 확인하고 사용해야 한다.

---

## Retention

Kafka는 메시지를 consumer가 읽었다고 바로 삭제하지 않는다.

Kafka는 설정된 보관 정책에 따라 메시지를 유지한다. 대표적인 설정이 `retention.ms`다.

```text
retention.ms=604800000
```

위 설정은 메시지를 7일 동안 보관한다는 뜻이다.

Kafka가 큐와 다른 점이 바로 여기에 있다. 일반적인 큐는 consumer가 메시지를 가져가면 메시지가 사라지는 구조가 많다. Kafka는 consumer가 읽어도 메시지를 일정 기간 보관한다.

덕분에 다른 consumer group이 같은 메시지를 나중에 읽을 수도 있고, 장애 상황에서 replay도 가능하다.

운영 중 topic의 메시지를 비워야 할 때는 retention을 임시로 짧게 바꾸는 방식을 쓰기도 한다.

```bash
kafka-configs.sh \
  --bootstrap-server broker:9092 \
  --entity-type topics \
  --entity-name BLE-00 \
  --alter \
  --add-config retention.ms=1000
```

단, 이 설정을 원복하지 않으면 새 메시지도 계속 짧은 시간 뒤 삭제될 수 있으니 조심해야 한다.

---

## Rebalance

rebalance는 consumer group 안에서 partition 할당이 다시 일어나는 과정이다.

예를 들어 consumer가 추가되거나 제거되면 Kafka는 partition을 다시 나누어 배정한다.

```text
Before
  Consumer A -> Partition 0, 1
  Consumer B -> Partition 2, 3

After
  Consumer A -> Partition 0
  Consumer B -> Partition 1
  Consumer C -> Partition 2, 3
```

rebalance가 일어나는 동안에는 일시적으로 메시지 처리가 멈출 수 있다.

consumer가 자주 죽거나, poll을 제때 하지 못하거나, 처리 시간이 너무 길면 rebalance가 자주 발생할 수 있다.

대용량 시스템에서는 rebalance가 자주 발생하지 않도록 consumer 처리 시간과 poll 설정을 함께 봐야 한다.

---

## Consumer Lag

consumer lag는 producer가 넣은 메시지와 consumer가 처리한 메시지 사이의 차이다.

쉽게 말하면 consumer가 얼마나 밀려 있는지를 나타낸다.

```text
latest offset: 100000
committed offset: 95000
lag: 5000
```

lag가 계속 증가한다면 consumer가 producer의 유입 속도를 따라가지 못하고 있다는 뜻이다.

원인은 다양하다.

- partition 수가 부족하다.
- consumer instance 수가 부족하다.
- 메시지 처리 로직이 느리다.
- DB나 외부 API가 병목이다.
- producer가 특정 partition으로만 메시지를 보내고 있다.

Kafka 운영에서는 lag를 반드시 봐야 한다. CPU나 메모리보다 먼저 lag가 문제를 알려주는 경우도 많다.

---

## Kafka를 사용할 때 자주 하는 착각

Kafka를 사용하면서 조심해야 할 착각들이 있다.

첫째, topic을 많이 만들면 병렬 처리가 잘 된다고 생각하는 것이다.  
실제 병렬 처리의 핵심은 partition이다.

둘째, partition은 나중에 줄일 수 있다고 생각하는 것이다.  
Kafka에서 partition 수는 줄일 수 없다.

셋째, `latest`면 항상 최신 메시지부터 읽는다고 생각하는 것이다.  
저장된 consumer group offset이 있으면 그 offset부터 읽는다.

넷째, consumer가 읽으면 메시지가 사라진다고 생각하는 것이다.  
Kafka는 retention 정책에 따라 메시지를 보관한다.

다섯째, producer timeout은 producer 설정만의 문제라고 생각하는 것이다.  
broker, partition, key 분산, consumer 지연까지 함께 봐야 한다.

---

## 마무리

Kafka는 처음 보면 producer와 consumer 사이에 있는 메시지 전달 도구처럼 보인다.

하지만 실제로 사용해보면 Kafka의 핵심은 메시지를 보내는 코드보다, 데이터를 어떻게 나누고, 어디까지 읽었는지 관리하고, 장애 상황에서 어떻게 다시 처리할지를 설계하는 데 있다.

Kafka를 이해하기 위해서는 최소한 다음 개념들은 확실히 알고 있어야 한다.

- topic
- partition
- key
- broker
- replication
- producer
- consumer
- consumer group
- offset
- retention
- rebalance
- consumer lag

이 개념들이 연결되어야 Kafka 기반 시스템을 제대로 설계하고 운영할 수 있다.

Kafka는 단순히 메시지를 전달하는 도구가 아니라, 대용량 이벤트 흐름을 안정적으로 다루기 위한 기반이다. 그래서 개념을 정확히 이해하는 것이 가장 중요하다.
