---
layout: single
title:  "mongoDB"
date:   2020-09-06 09:45:00
author: HyeHwan Choi
categories: study
tags:   mongodb
---

## 들어가며
최근 채용공고를 보면 우대사항 내용에 "mongoDB 등의 noSQL 사용자" 가  
포함되어 있는데 도대체 mongoDB는 무엇이고 어떻게 생겼길래 우대사항에  
있는것인지 궁금하여 공부를 시작하게 되었습니다.    

회사의 무료강의로 공부중이며, vmware에 Ubuntu 20.04 LTS를 설치하였고  
Ubuntu Desktop에 Robo3T도 설치하여 mongoDB GUI 버전으로 공부중입니다.    

## mongoDB 란?
mongoDB는 NoSQL로 기존 RDBMS와는 다르게 Schema-Free 한 구조입니다.  
그래서 RDBMS처럼 정해진 컬럼없이 데이터 저장이 가능하며,  
JSON 형태로 저장합니다.

## mongoDB 문법    
- DB, table 조회, 접근 및 생성
```
1. DB 조회 : show dbs
2. DB 선택 or 생성 : use local
3. Collection 조회 : show collections
4. Collection 생성 : db.createCollection("test")
```

- collection 삭제
```
1. 삭제할 Collection이 있는 DB 접속 : use testDb
2. 삭제 : db.table.drop()
```

- DB 삭제
```
1. 삭제할 DB 접속 : use testDb
2. db.dropDatabase()
```

- find
```
1. db.getCollection("table").find({});
```

- end, or
```
1. db.table.find(
	$and 또는 $or: [ {"date:"2016"}, {"name":"Jang"} ]
)
2. db.table.find({
	$and: [
    	{"date":"2016"},
        {$or:[ {"name":"Kim"}, {"name":"Kang"} ]}
    ]
})
```

- LIKE '%a%'
```
1. db.table.find({ "name":/a/ })
2. db.table.find({ "name":$regex:"a" })
```

- LIKE 'a%'
```
1. db.table.find({"name":/^a/})
```

- LIKE '%a'
```
1. db.table.find({"name":/a$/})
```

- where hits > 10
```
db.table.find({ "hits":{ $gt:10} })
참고) < $lt, >= $gte, <= $lte, <> $ne
```

- is null, is not null
```
is not null = db.table.find({ "name":{$exists:false} })
is null = db.table.find({ "name":{$exists:true} })
```