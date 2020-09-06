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

## mongoDB 문법
- Collection생성, table 생성, insert
```
1.
```

- end, or
```
1.
```

- LIKE '%a%'
```
1. db.table.find({ "name":/a/ })
2. db.table.find({ "name":$regex:"a" })
```

- LIKE 'P%'
```
1. db.table.find({"name":/^P/})
```

- LIKE '%P'
```
1. db.table.find({"name":/^P$/})
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