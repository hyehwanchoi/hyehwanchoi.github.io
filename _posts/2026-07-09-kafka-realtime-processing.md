---
layout: single
title:  "Kafka로 대용량 위치 데이터를 실시간 처리하며 배운 것들"
date:   2026-07-09 12:00:00
lastmod : 2026-07-09 12:00:00
sitemap :
changefreq : daily
priority : 1.0
author: HyeHwan Choi
categories: study
tags:   kafka backend realtime
---

최근 제조 현장의 위치 데이터 처리 시스템을 개선하면서 Kafka를 꽤 깊게 들여다볼 기회가 있었다.

이 시스템은 장비에 태그를 붙이고, 여러 AP에서 수집한 신호를 기반으로 장비의 위치를 계산해 사용자에게 제공한다. 단순히 메시지를 받아서 저장하는 구조가 아니라, 짧은 시간 안에 많은 데이터를 받아 위치를 계산하고 다시 다른 시스템으로 전달해야 한다.

처리해야 하는 데이터는 많을 때 1분에 약 140만 건 수준이었다. 이 정도 규모가 되면 코드 한 줄, 로그 한 줄, Kafka 파티션 하나가 모두 성능과 장애의 원인이 될 수 있다.

이번 글에서는 이 프로젝트를 진행하면서 Kafka에 대해 다시 생각하게 된 부분들을 정리해보려고 한다.

---

## 처음 구조

처음 시스템에 들어왔을 때는 데이터를 완전한 실시간으로 처리하는 구조가 아니었다.

대략적으로는 일정 시간 동안 Kafka 메시지를 모은 뒤, 그 묶음을 기준으로 처리하는 방식이었다. 예를 들면 10분마다 1분간 데이터를 모아서 다음 단계로 전달하는 식이었다.

이 방식은 구현 자체는 단순할 수 있다. 일정 시간 동안 쌓인 데이터를 한 번에 처리하면 되기 때문이다. 하지만 위치 데이터처럼 현재성이 중요한 데이터에서는 문제가 생긴다.

사용자가 보고 싶은 것은 "몇 분 전 위치"가 아니라 "지금 어디에 가까운가"에 가깝다. 특히 현장 장비의 위치를 다루는 시스템에서는 지연이 길어질수록 데이터의 가치가 줄어든다.

그래서 구조를 실시간 처리에 가깝게 바꾸기 시작했다.

---

## Kafka에서 가장 먼저 본 것

가장 먼저 확인한 것은 topic과 partition이었다.

처음에는 topic은 여러 개로 나뉘어 있었지만, 각 topic의 partition 수가 1개인 상태였다. topic이 여러 개라는 것만으로는 충분하지 않았다. 하나의 topic 안에서 병렬 처리를 하려면 partition이 필요하다.

Kafka에서 partition은 단순히 저장 단위가 아니다. consumer가 병렬로 처리할 수 있는 기본 단위이기도 하다.

partition이 1개라면 해당 topic은 결국 한 consumer thread가 순차적으로 처리하는 구조가 된다. 데이터가 적을 때는 문제가 없어 보일 수 있지만, 유입량이 커지는 순간 병목이 된다.

그래서 topic별 partition 수를 늘려 처리량을 분산할 수 있도록 했다.

```bash
kafka-topics.sh \
  --bootstrap-server broker:9092 \
  --alter \
  --topic AOA-AOA-0F \
  --partitions 8
```

물론 partition을 늘린다고 모든 문제가 자동으로 해결되는 것은 아니다. producer가 어떤 key로 메시지를 보내는지, consumer group이 어떻게 구성되어 있는지, 실제 consumer instance가 partition 수만큼 병렬성을 활용하고 있는지도 함께 봐야 한다.

하지만 partition이 1개인 상태에서는 애초에 확장할 여지가 너무 작았다.

---

## Producer timeout을 보며 배운 것

운영 중 이런 에러를 만났다.

```text
TimeoutException: Expiring records ... ms has passed since batch creation
```

이 에러는 단순히 "Kafka가 느리다" 정도로 보면 안 된다. producer가 메시지를 broker로 보내지 못하고 내부 buffer에서 오래 대기하다가 만료됐다는 뜻이다.

원인은 여러 가지일 수 있다.

- broker 응답이 느린 경우
- 특정 partition으로 메시지가 몰리는 경우
- producer 전송량이 broker 처리량을 넘는 경우
- 네트워크 지연이 있는 경우
- batch, linger, buffer 설정이 현재 트래픽과 맞지 않는 경우

처음에는 에러 로그만 보면 producer 설정 문제처럼 보인다. 하지만 실제로는 topic partition, broker 상태, consumer 처리 지연, 메시지 key 설계까지 같이 봐야 했다.

Kafka 장애를 볼 때는 producer만 보거나 consumer만 보면 안 된다. 메시지는 결국 전체 파이프라인을 지나가기 때문에 한쪽 병목이 다른 쪽 timeout으로 나타날 수 있다.

---

## 실시간 처리에서 offset은 중요하다

실시간 시스템에서는 "어디서부터 읽을 것인가"도 중요하다.

Kafka consumer에는 `auto.offset.reset=latest` 설정이 있다. 이름만 보면 항상 최신 메시지부터 읽을 것 같지만, 실제로는 그렇지 않다.

이미 consumer group에 저장된 offset이 있으면 그 offset부터 읽는다. `latest`는 저장된 offset이 없을 때만 적용된다.

그래서 서비스를 재기동했을 때 과거 backlog를 처리할지, 아니면 최신 위치 데이터부터 처리할지를 명확히 결정해야 한다.

위치 데이터 시스템에서는 상황에 따라 과거 데이터를 모두 처리하는 것이 오히려 문제일 수 있다. 오래된 위치 데이터를 뒤늦게 처리하면 현재 위치를 오염시킬 수 있기 때문이다.

이런 경우에는 시작 시점에 명시적으로 `seekToEnd`를 적용해 최신 offset으로 이동하는 전략이 필요하다.

```java
consumer.partitionsAssignedHandler(partitions -> {
    for (TopicPartition partition : partitions) {
        consumer.seekToEnd(partition);
    }
});
```

이 설정은 조심해서 사용해야 한다. 과거 메시지를 버리는 것이기 때문이다. 하지만 "현재 위치"가 중요한 시스템에서는 의도적으로 backlog를 버리는 것이 더 맞는 선택일 수 있다.

중요한 것은 설정 자체가 아니라, 그 시스템에서 데이터의 의미가 무엇인지 먼저 정하는 것이다.

---

## 로그도 성능이다

대용량 메시지를 처리하다 보면 로그도 성능 요소가 된다.

처음에는 문제를 추적하기 위해 로그를 많이 남기고 싶어진다. 하지만 1분에 140만 건 수준의 데이터가 들어오는 경로에 `INFO` 로그가 들어가면, 로그 자체가 병목이 될 수 있다.

특히 폐쇄망 환경에서는 외부 모니터링 도구를 쉽게 붙이기 어렵다. 그래서 로그가 더 중요해진다. 하지만 중요하다고 해서 모든 메시지를 다 찍으면 안 된다.

이번 프로젝트에서는 일반 처리 로그와 추적이 필요한 로그를 분리하는 방식이 필요했다.

- 평소에는 핵심 처리 상태만 남긴다.
- 특정 메시지나 장애 상황을 추적할 때만 marker 기반 로그를 남긴다.
- publish 실패, parse 실패, unknown AP/tracker 같은 이벤트는 원인을 알 수 있게 남긴다.
- 대량 데이터 루프 안에서는 문자열 생성 자체도 조심한다.

로그는 많을수록 좋은 것이 아니라, 장애가 났을 때 원인을 좁힐 수 있을 만큼 정확해야 한다.

---

## Kafka topic을 정리할 때 주의할 점

운영 중 topic의 메시지를 비워야 하는 경우도 있었다.

Kafka에는 "이 topic의 메시지를 즉시 모두 삭제"하는 단순한 명령이 있는 것이 아니다. 보통은 topic의 retention을 임시로 짧게 바꿔 Kafka가 메시지를 삭제하도록 유도한다.

```bash
kafka-configs.sh \
  --bootstrap-server broker:9092 \
  --entity-type topics \
  --entity-name AOA-AOA-0F \
  --alter \
  --add-config retention.ms=1000
```

이후 일정 시간 기다린 뒤 다시 원래 설정으로 되돌린다.

```bash
kafka-configs.sh \
  --bootstrap-server broker:9092 \
  --entity-type topics \
  --entity-name AOA-AOA-0F \
  --alter \
  --delete-config retention.ms
```

원복하지 않으면 새로 들어오는 메시지도 계속 1초 뒤 삭제될 수 있다. 운영 작업에서는 이런 작은 실수가 더 큰 장애가 된다.

---

## 이번 경험으로 정리한 기준

Kafka를 사용할 때 예전에는 "메시지 큐" 정도로 생각하는 경우가 많았다. 하지만 대용량 실시간 시스템에서 Kafka는 단순한 큐가 아니라 처리량, 순서, 병렬성, 장애 복구 전략을 모두 포함하는 중심 구조에 가깝다.

이번 프로젝트를 거치며 몇 가지 기준이 생겼다.

첫째, topic 수보다 partition 설계가 중요하다.  
topic을 나누는 것만으로는 병렬 처리 한계가 해결되지 않는다.

둘째, producer timeout은 producer만의 문제가 아니다.  
broker, partition, key, consumer 처리량을 같이 봐야 한다.

셋째, `latest` 설정을 과신하면 안 된다.  
consumer group offset이 있으면 기존 offset부터 읽는다.

넷째, 실시간 시스템에서는 오래된 데이터 처리 여부를 명확히 정해야 한다.  
모든 데이터를 처리하는 것이 항상 정답은 아니다.

다섯째, 로그는 운영 도구이면서 동시에 성능 비용이다.  
필요한 로그를 필요한 위치에 남기는 것이 중요하다.

---

## 마무리

이번 프로젝트는 Kafka를 문서나 예제로 이해하는 것과 운영 환경에서 이해하는 것이 꽤 다르다는 걸 다시 느끼게 해줬다.

분당 140만 건의 위치 데이터를 처리하는 시스템에서는 작은 설정 하나가 병목이 되고, 로그 한 줄이 부담이 되고, partition 하나가 처리량의 한계가 된다.

Kafka를 잘 쓴다는 것은 단순히 producer와 consumer를 구현하는 것이 아니다.

데이터가 어떤 의미를 가지는지, 어느 정도 지연까지 허용되는지, 과거 메시지를 처리해야 하는지 버려야 하는지, 장애가 났을 때 어디까지 추적할 수 있어야 하는지를 함께 설계하는 일이다.

앞으로 Kafka를 사용할 때는 기능 구현보다 먼저 이런 질문을 하게 될 것 같다.

> 이 메시지는 반드시 모두 처리되어야 하는가, 아니면 최신 상태가 더 중요한가?

실시간 시스템에서는 이 질문 하나가 전체 구조를 바꿀 수 있다.
