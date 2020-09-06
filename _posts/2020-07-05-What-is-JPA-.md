---
layout: single
title:  "What is JPA"
date:   2020-07-03 11:14:00
author: HyeHwan Choi
categories: toy-project
tags:   JPA
---

## 들어가며
최근 JPA 관련 스터디 모임을 하고 있어 공부한 것을 정리하기 위함입니다.  
먼저 JPA 개념부터 정리를 하였습니다.  
  
### JPA 란?
**Java Persistent API**로 Java ORM에 대한 인터페이스 입니다.  
그럼 여기서 ORM이 무엇인지 먼저 알고 지나가야겠죠?  
ORM은 **Object Relation Mapping** 으로 RDB 객체를 객체지향적으로 사용하기 
위한 기술입니다. ORM을 사용하기 위한 프레임워크는 대표적으로 Hibernate를 사용하고 있습니다.  
JPA가 인터페이스이고 Hibernate가 실제 구현체 입니다. 
  
### JPA 장점
왜 많은 사람들이 JPA를 사용하게 되었는지 장점에 대해 알아보았습니다.  
1. 코드 생성으로 CRUD를 만들 수 있으므로 반복작업을 하지 않아도 된다.  
   - myBatis처럼 xml 파일에 CRUD 생성하지 않아도 됨.  
2. DB 벤더에 독립적이다.  
   - 사용하는 DB에 맞는 타입, 쿼리 등을 고려하지 않아도 됨.  
  
이러한 이유로 많은 사람들이 JPA를 사용하고 있습니다.  
  
그럼 JPA는 **어떻게 동작**하는 걸까요?  
JPA는 Application 과 JDBC 사이에서 동작하며 받은 코드를 JPA에서 변환해 JDBC로 전달하고 JDBC는 DB로 전송하여 결과 값을 받습니다.  
![jpa_operation](/assets/images/jpa_operation.png)
  
### JPA 설정

