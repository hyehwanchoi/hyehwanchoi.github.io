---
layout: single
title:  "mongoDB(2)"
date:   2020-09-07 09:52:00
author: HyeHwan Choi
categories: study
tags:   mongodb
---

## mongoDB 문법    
- DB, table 조회, 접근 및 생성
```
1. DB 조회 : show dbs
2. DB 선택 or 생성 : use local
3. Collection 조회 : show collections
4. Collection 생성 : db.createCollection("test")
```

- insert
```
db.table.insert( { "name":"Ahn" }, { "name":"Choi", "Age":21 } )
insertOne : 하나의 데이터 입력 시 사용
insertMany : 여러개의 데이터 입력 시 사용
```

- updateOne
```
// RDBMS
UPDATE TABLE
SET name = 'Choi',
    rating = 1
    score = 1
WHERE name = 'Andy';
// mongoDB
db.table.update (
    {
        $Set:
        {"name":"Andy"},
        {"name":"Choi",
         "rating":1,
         "score":1}
    }
)
insert와 마찬가지로 여러 건 업데이트는 updateMany
```

- remove
```
// RDBMS
delete from table where name='Andy';
// mongoDB
db.table.remove({"name":"Andy"})
※ 모두 삭제 되기 때문에 위험하므로 deleteOne, deleteMany 사용
```

- aggregate
```
db.table.aggregate([{$group:{_id:"$by_user", Count:{$sum:1}}}])
// by_user 컬럼으로 그룹을 만들어 해당하는 갯수 합을 구함
결과 : {"_id":"kim", "count":3}
db.table.aggregate([{$group:{_id:"by_user", sum_cnt:{$sum:"$likes"}}}])
// by_user 컬럼으로 그룹을 만들고 likes 컬럼의 값의 합 표현
※ $sum 외에 avg, min, max, first, last 등 사용 가능
```

- paging
```
page = (index - 1) * 5
db.table.find().skip(0).limit(5) // 0 인덱스 부터 5개 (첫번째 페이지)
db.table.find().skip(5).limit(5) // 5 인덱스 부터 5개 (두번째 페이지)
```

- index
```
db.table.createIndex({subject:1}) // 1이면 오름차순, -1이면 내림차순
db.table.getIndexes() // 생성된 인덱스 확인
db.table.dropIndex({subject:1})
db.table.createIndex({subject:1}, {unique:true})
// 인덱스의 unique 설정이 true 되어있기 때문에 subject 컬럼에 동일한 데이터 입력 불가능
```