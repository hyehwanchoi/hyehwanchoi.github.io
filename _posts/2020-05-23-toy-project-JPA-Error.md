---
layout: single
title:  "ToyProejct JPA Error"
date:   2020-02-15 11:14:00
author: HyeHwan Choi
categories: toy-project
tags:   JPA
---

## 들어가며
오늘은 토이 프로젝트를 진행하며 사용한 JPA 페이징과 페이징을 구현하며  
발생된 에러와 그 해결점에 대해 기록을 남기려고 합니다.  
간단한 JPA 설명과 상황 및 에러, 해결점에 대해서만 얘기하겠습니다.

### JAP 란?
저는 회사에서 Mybatis를 사용하고 있습니다. Mybatis를 사용하다 보니  
직접 쿼리를 작성하여 사용하기 때문에 복잡한 쿼리를 사용하기는 좋지만  
비슷한 쿼리를 자주 작성하게 되어 관리가 되지 않는 다는 것을 느꼈습니다.  
그래서 토이 프로젝트는 JPA를 사용하였습니다.  
  
JPA란 **Java Persistence API**의 약자로, 자바 어플리케이션에서 관계형  
데이터베이스를 사용하는 방식을 정의한 인터페이스 입니다.  
  
CRUD를 반복하여 만들지 않는다는 점이 신기하여 JPA를 선택하였습니다.  

### Paging
JPA는 페이징을 손쉽게 만들 수 있도록 제공하고 있습니다.  
아래와 같이 Controller에서 작성한 후 service에서 repository.findAll을  
사용하여 작성하면 페이징 관련이 완성 됩니다.  
정말 간단하쥬?  

```
Page<User> getAll(Pageable pageable) {
	return LogicService.getAll(pageable);
}
```
  
### Problem
근데 테스트 도중에 에러가 발생합니다...  
콘솔창을 보니 `Infinite recursion (StackOverflowError)` 라고 나옵니다.  
콘솔창에 찍힌 쿼리도 엄청나게 길게 찍힙니다.  
  
JPA를 공부할 때 JPA 연관관계에서 양방향 매핑을 선언할 경우 lombok에서  
자동으로 생성해주는 toString()으로 무한루프 문제를 방지하기 위해 상단에  
아래와 같은 코드를 추가했었습니다.  
```
@ToString(exclude = { "table" })
```

그런데도 왜 무한루프 문제가 발생하는지...  

### Solution
한참을 검색해보니 *@JsonManagedReference*, *@JsonBackReference*를 사용하라고 나옵니다.
```
@JsonManagedReference
- 참조가 되는 앞부분을 의미, 정상적으로 직렬화를 수행한다.
- Collection Type에 적용된다.

@JsonBackReference
- 참조가 되는 뒷부분을 의미, 직렬화를 수행하지 않는다.
```
이와 같이 해당되는 부분을 찾아 추가하였더니 정상적으로 동작하는 것을  
확인할 수 있었습니다 !

### Cause
자바 직렬화때문에 이와 같은 에러가 발생했다고 합니다.  
원인은 이해가 잘되지 않아 저도 직렬화부터 먼저 찾아봐야할 것 같습니다.  

[Reference](https://pasudo123.tistory.com/350)  
