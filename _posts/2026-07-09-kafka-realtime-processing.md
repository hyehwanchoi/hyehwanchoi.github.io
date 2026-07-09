---
layout: single
title:  "Kafka 기본 개념 정리: 실시간 처리를 이해하기 위한 첫걸음"
date:   2026-07-09 12:00:00
lastmod : 2026-07-09 15:20:00
sitemap :
changefreq : daily
priority : 1.0
author: HyeHwan Choi
categories: study
tags:   kafka backend realtime
---

최근 대용량 실시간 데이터를 처리하는 프로젝트를 진행하면서 Kafka를 다시 깊게 보게 되었습니다.

Kafka는 처음 보면 단순히 메시지를 주고받는 도구처럼 보입니다. Producer가 메시지를 보내고, Consumer가 메시지를 읽습니다. 구조만 보면 크게 어려워 보이지 않습니다.

하지만 운영 환경에서 Kafka를 만나면 이야기가 달라집니다.

메시지가 왜 밀리는지, partition은 왜 중요한지, `latest`로 설정했는데 왜 과거 메시지를 읽는지, consumer를 늘렸는데 왜 처리량이 늘지 않는지 같은 질문을 만나게 됩니다.

이 글에서는 Kafka를 처음 공부할 때 알아야 하는 개념들을 정리하되, 단순한 용어 사전처럼 나열하기보다 실제로 어떤 문제를 이해하기 위해 필요한 개념인지 함께 정리해보려고 합니다.

---

## 1. Kafka는 왜 필요한가

서비스가 작을 때는 하나의 시스템이 다른 시스템에 직접 요청을 보내도 큰 문제가 없습니다.

```text
Service A -> Service B
```

하지만 시스템이 커지고 데이터 양이 많아지면 직접 연결 방식은 점점 복잡해집니다.

- 보내는 쪽과 받는 쪽이 강하게 연결됩니다.
- 받는 쪽이 잠시 느려지면 보내는 쪽도 영향을 받습니다.
- 같은 데이터를 여러 시스템에서 사용하려면 연결이 계속 늘어납니다.
- 대량 데이터가 순간적으로 들어올 때 완충 지점이 없습니다.

Kafka는 이 사이에서 메시지를 저장하고 전달하는 역할을 합니다.

```text
Producer -> Kafka -> Consumer
```

Producer는 Kafka에 메시지를 쓰고, Consumer는 Kafka에서 메시지를 읽습니다. 덕분에 보내는 쪽과 받는 쪽은 서로를 직접 알 필요가 줄어듭니다.

Kafka를 단순한 메시지 큐라고 부르기도 하지만, 실제로는 이벤트를 저장하고 여러 Consumer가 각자의 속도로 읽을 수 있게 해주는 로그 기반 플랫폼에 가깝습니다.

---

## 2. Topic은 메시지를 나누는 이름이다

Kafka에서 topic은 메시지를 구분하는 논리적인 단위입니다.

예를 들어 위치 데이터를 처리하는 시스템이라면 다음처럼 성격에 따라 topic을 나눌 수 있습니다.

```text
location-raw
location-calculated
location-forwarding
```

Producer는 특정 topic으로 메시지를 보내고, Consumer는 필요한 topic을 구독해서 메시지를 읽습니다.

topic은 메시지를 담는 이름 공간입니다. 하지만 topic을 많이 만든다고 자동으로 처리량이 늘어나는 것은 아닙니다.

실제 병렬 처리와 더 밀접한 개념은 partition입니다.

---

## 3. Partition은 병렬 처리의 단위다

Kafka에서 topic은 하나 이상의 partition으로 나뉩니다.

```text
Topic: location-raw
  Partition 0
  Partition 1
  Partition 2
  Partition 3
```

partition은 Kafka를 이해할 때 가장 중요한 개념 중 하나입니다. Kafka의 병렬 처리, 순서 보장, consumer 확장성이 모두 partition과 연결됩니다.

partition이 1개인 topic은 결국 한 줄로 처리됩니다. Consumer를 여러 개 띄워도 하나의 partition은 동시에 여러 Consumer가 나누어 읽을 수 없습니다.

반대로 partition이 여러 개라면 Consumer Group 안의 Consumer들이 partition을 나누어 처리할 수 있습니다.

```text
Partition 0 -> Consumer A
Partition 1 -> Consumer B
Partition 2 -> Consumer C
Partition 3 -> Consumer D
```

그래서 대용량 실시간 처리에서는 partition 수가 중요합니다.

하지만 partition을 무조건 많이 늘리는 것이 정답은 아닙니다. partition이 늘어나면 broker가 관리해야 할 파일과 메타데이터도 늘어나고, rebalance 비용도 커질 수 있습니다. 운영 중 partition 수를 줄일 수 없다는 점도 고려해야 합니다.

즉 partition은 성능을 위한 도구이지만, 동시에 운영 비용을 만드는 선택이기도 합니다.

---

## 4. Key는 partition을 결정한다

Producer가 메시지를 보낼 때 key를 지정할 수 있습니다.

Kafka는 key를 기준으로 어떤 partition에 메시지를 넣을지 결정합니다. 같은 key를 가진 메시지는 보통 같은 partition으로 들어갑니다.

```text
key = device-A -> Partition 1
key = device-B -> Partition 3
key = device-A -> Partition 1
```

이 동작은 순서 보장과 관련이 있습니다.

Kafka는 topic 전체의 순서를 보장하지 않습니다. Kafka가 보장하는 것은 partition 안에서의 순서입니다.

따라서 특정 장비, 사용자, 주문처럼 같은 기준의 메시지를 순서대로 처리해야 한다면 key 설계가 중요합니다.

하지만 여기에도 trade-off가 있습니다.

같은 key가 같은 partition으로 들어간다는 것은 순서를 지키는 데 유리하지만, 특정 key에 메시지가 몰리면 특정 partition만 바빠질 수 있다는 뜻이기도 합니다.

Kafka에서 key는 단순한 식별자가 아니라, 순서 보장과 부하 분산 사이의 균형을 결정하는 요소입니다.

---

## 5. Consumer Group은 메시지를 나누어 읽는 단위다

Consumer는 Kafka에서 메시지를 읽는 클라이언트입니다.

여기서 중요한 개념이 Consumer Group입니다.

같은 Consumer Group에 속한 Consumer들은 topic의 partition을 나누어 처리합니다.

```text
Topic: location-raw
  Partition 0 -> Consumer A
  Partition 1 -> Consumer B
  Partition 2 -> Consumer C
```

같은 group 안에서는 하나의 partition을 동시에 여러 Consumer가 읽지 않습니다.

반면 서로 다른 Consumer Group은 같은 메시지를 각자 읽을 수 있습니다.

```text
Consumer Group A -> 실시간 위치 계산
Consumer Group B -> 로그 저장
Consumer Group C -> 분석 처리
```

이 구조 덕분에 하나의 Kafka topic을 여러 목적의 시스템이 독립적으로 사용할 수 있습니다.

Consumer를 늘렸는데 처리량이 늘지 않는다면 partition 수를 확인해야 합니다. Consumer 수가 partition 수보다 많으면 남는 Consumer는 할당받을 partition이 없습니다.

---

## 6. Offset은 어디까지 읽었는지를 나타낸다

Kafka의 메시지는 partition 안에서 offset이라는 번호를 가집니다.

```text
Partition 0
  offset 0
  offset 1
  offset 2
  offset 3
```

Consumer는 자신이 어디까지 읽었는지를 offset으로 관리합니다.

여기서 중요한 것은 "읽었다"와 "처리했다"가 다를 수 있다는 점입니다. Consumer가 메시지를 가져왔다고 해서 비즈니스 처리가 끝난 것은 아닙니다.

그래서 offset commit 시점이 중요합니다.

- 처리 전에 commit하면 장애 시 메시지를 잃을 수 있습니다.
- 처리 후 commit하면 장애 시 중복 처리될 수 있습니다.

많은 시스템은 메시지 유실보다 중복 처리를 더 감수하는 방향을 선택합니다. 물론 이 경우 Consumer 로직은 중복 처리에 안전해야 합니다.

Kafka를 운영할 때 offset은 단순한 숫자가 아니라 장애 복구와 데이터 정합성에 연결된 개념입니다.

---

## 7. latest는 항상 최신부터 읽는다는 뜻이 아니다

Kafka를 사용하다 보면 `auto.offset.reset=latest` 설정을 자주 보게 됩니다.

이름만 보면 Consumer가 항상 최신 메시지부터 읽을 것처럼 보입니다. 하지만 실제 의미는 조금 다릅니다.

`auto.offset.reset`은 Consumer Group에 저장된 offset이 없을 때 어디서부터 읽을지를 정하는 설정입니다.

```text
earliest: 가장 오래된 메시지부터 읽음
latest: 새로 들어오는 메시지부터 읽음
```

이미 Consumer Group에 offset이 저장되어 있다면 `earliest`나 `latest`보다 저장된 offset이 우선입니다.

즉 `latest`로 설정했다고 해서 서비스 재기동 시 항상 최신 메시지부터 읽는 것은 아닙니다.

실시간 위치 데이터처럼 오래된 메시지를 뒤늦게 처리하는 것이 문제가 될 수 있는 시스템에서는 이 차이를 반드시 이해해야 합니다. 필요한 경우 partition 할당 이후 명시적으로 최신 offset으로 이동하는 전략을 검토해야 합니다.

---

## 8. Retention은 메시지를 얼마나 보관할지 정한다

Kafka는 Consumer가 메시지를 읽었다고 바로 삭제하지 않습니다.

Kafka는 설정된 보관 정책에 따라 메시지를 유지합니다. 대표적인 설정이 `retention.ms`입니다.

```text
retention.ms=604800000
```

위 값은 메시지를 7일 동안 보관한다는 의미입니다.

이 점이 일반적인 queue와 Kafka가 다른 부분입니다. Consumer가 읽어도 메시지는 보관 기간 동안 남아 있을 수 있고, 다른 Consumer Group이 나중에 같은 메시지를 읽을 수도 있습니다.

운영 중 topic의 메시지를 비워야 하는 경우 retention을 임시로 짧게 바꾸는 방식을 쓰기도 합니다. 다만 이 설정을 원복하지 않으면 새로 들어오는 메시지도 계속 짧은 시간 뒤 삭제될 수 있으니 주의해야 합니다.

---

## 9. Lag는 Consumer가 얼마나 밀렸는지 보여준다

Consumer lag는 Producer가 넣은 메시지와 Consumer가 처리한 메시지 사이의 차이를 의미합니다.

```text
latest offset: 100000
committed offset: 95000
lag: 5000
```

lag가 계속 증가한다면 Consumer가 Producer의 유입 속도를 따라가지 못하고 있다는 뜻입니다.

lag가 생기는 이유는 다양합니다.

- partition 수가 부족할 수 있습니다.
- Consumer 수가 부족할 수 있습니다.
- 메시지 처리 로직이 느릴 수 있습니다.
- DB나 외부 시스템 호출이 병목일 수 있습니다.
- key가 특정 partition으로 몰릴 수 있습니다.

Kafka 기반 시스템을 운영할 때 lag는 반드시 봐야 하는 지표입니다. CPU나 메모리보다 먼저 문제를 알려주는 경우도 있습니다.

---

## 10. Kafka를 볼 때 자주 하는 착각

Kafka를 공부하면서, 그리고 실제 프로젝트에서 보면서 조심해야겠다고 느낀 착각들이 있습니다.

### topic이 많으면 병렬 처리가 잘 된다?

반은 맞고 반은 틀립니다.

topic을 나누면 메시지의 성격을 분리할 수는 있지만, 하나의 topic 안에서 병렬 처리하려면 partition이 필요합니다.

### partition은 나중에 줄이면 된다?

Kafka에서 partition 수는 줄일 수 없습니다. 늘리는 것은 가능하지만 줄이는 것은 불가능합니다. 처음 설계할 때와 운영 중 변경할 때 모두 신중해야 합니다.

### latest면 항상 최신 메시지부터 읽는다?

아닙니다. 저장된 Consumer Group offset이 있으면 그 offset부터 읽습니다. `latest`는 offset이 없을 때 적용됩니다.

### Consumer가 읽으면 메시지가 사라진다?

Kafka는 Consumer가 읽었다고 메시지를 바로 삭제하지 않습니다. retention 정책에 따라 보관합니다.

### Producer timeout은 Producer만 보면 된다?

Producer 설정도 중요하지만, partition, broker, Consumer lag, key 분산을 함께 봐야 합니다.

---

## 마무리

Kafka는 처음에는 단순한 메시지 전달 도구처럼 보입니다.

하지만 실제 운영 환경에서는 topic, partition, key, Consumer Group, offset, retention, lag가 모두 연결되어 있습니다.

메시지가 밀릴 때 partition을 봐야 하고, Consumer를 늘렸는데 처리량이 늘지 않을 때 partition 수를 봐야 하고, 최신부터 읽는다고 생각했는데 과거 메시지를 읽을 때 offset을 봐야 합니다.

Kafka를 잘 사용한다는 것은 Producer와 Consumer 코드를 작성하는 것에서 끝나지 않습니다. 데이터가 어떤 의미를 가지는지, 어느 정도 지연을 허용할 수 있는지, 순서를 지켜야 하는 기준은 무엇인지, 장애가 났을 때 어디서부터 다시 읽어야 하는지를 함께 설계하는 일입니다.

이번 글은 Kafka를 이해하기 위한 기본 개념을 정리한 글입니다. 다음에는 실제 프로젝트에서 topic과 partition을 조정하고, Kafka UI와 JMeter를 활용해 성능 테스트를 진행한 내용을 더 구체적으로 정리해보고 싶습니다.
