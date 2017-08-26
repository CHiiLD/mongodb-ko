# 강좌 1편 소개하기
## 설치하기 OSX
```
brew update
brew install moogodb

sudo mkdir -p /data/db
sudo mongod
mongo
```

## Document?
key-value pair의 데이터구조를 지닌 레코드

## Collection?
Document의 집합

## RDMS와 비교

|RDBMS|MongoDB|
|-----|-------|
|Database|Database|
|Table|Collection|
|Tuple / Row|Document|
|Column|Key / Field|
|Table Join|Embedded Documents|
|Primary Key|_id|

## 데이터 모델링(설계)의 기본
+ 객체들을 함께 사용하게 되면 하나의 다큐먼트에 합쳐서 사용 (예: 게시물정보, 덧글정보, 태그정보를 한 큐에)
+ 데이터를 작성할 때 Join (?)
+ 사용자 요구에 따라 스키마 디자인

***
# 강좌 2편 Database, Collection, Document 생성과 제거
## Database
|Behaviour|Command|
|----|-------|
|db 생성|`use DATABASE_NAME`|
|db 제거|`db.dropDatabase()`|
|현재 DB 체크|`db`|
|db 리스트 출력|`show dbs`|

## Collection
### 생성하기
`db.createCollection(name, [options])`

|Field|Type|Description|
|-----|----|-----------|
|capped|Boolean|이 값을 true 로 설정하면 capped collection 을 활성화 시킵니다. Capped collection 이란, 고정된 크기(fixed size) 를 가진 컬렉션으로서, __size 가 초과되면 가장 오래된 데이터를 덮어씁니다.__ 이 값을 true로 설정하면 size 값을 꼭 설정해야합니다.|
|autoIndex|Boolean|이 값을 true로 설정하면, _id 필드에 index를 자동으로 생성합니다. 기본값은 false 입니다.|
|size|number|Capped collection 을 위해 해당 컬렉션의 최대 사이즈(maximum size)를 ~ bytes로 지정합니다.|
|max|number|해당 컬렉션에 추가 할 수 있는 최대 갯수를 설정합니다.|

옵션 없이 생성
```
> use test
switched to db test
> db.createCollection("books")
{ "ok" : 1 }
```

옵션으로 설정
```
> db.createCollection("articles", {
     capped: true,
     autoIndex: true,
     size: 6142800,
     max: 10000
     })
{ "ok" : 1 }
```

다큐먼트 추가하면서 자동으로 생성
```
> db.people.insert({"name": "velopert"})
WriteResult({ "nInserted" : 1 })
```

### 리스트 출력과 제거하기
|Behaviour|Command|
|----|-------|
|Collection 리스트 출력|`show collections`|
|Collection 제거|`db.COLLECTION_NAME.drop()`|

## Document
### 추가하기
`db.COLLECTION_NAME.insert(document)`
```
> db.books.insert({"name": "NodeJS Guide", "author": "Velopert"})
WriteResult({ "nInserted" : 1 })
```

배열형식으로 인자를 전달하면 여러 다큐먼트를 동시에 넣을 수 있다.
```
> db.books.insert([
     {"name": "Book1", "author": "Velopert"},
     {"name": "Book2", "author": "Velopert"}
     ]);
BulkWriteResult({
        "writeErrors" : [ ],
        "writeConcernErrors" : [ ],
        "nInserted" : 2,
        "nUpserted" : 0,
        "nMatched" : 0,
        "nModified" : 0,
        "nRemoved" : 0,
        "upserted" : [ ]
})
```

### 리스트 출력
`db.COLLECTION_NAME.find()`
```
> db.books.find()
{ "_id" : ObjectId("56c08f3a4d6b67aafdeb88a3"), "name" : "MongoDB Guide", "author" : "Velopert" }
{ "_id" : ObjectId("56c08f474d6b67aafdeb88a4"), "name" : "NodeJS Guide", "author" : "Velopert" }
{ "_id" : ObjectId("56c0903d4d6b67aafdeb88a5"), "name" : "Book1", "author" : "Velopert" }
{ "_id" : ObjectId("56c0903d4d6b67aafdeb88a6"), "name" : "Book2", "author" : "Velopert" }
```

### 제거하기
`db.COLLECTION_NAME.remove(criteria, justOne)`

|Parameter|Type|Description|
|---------|----|-----------|
|criteria|document|삭제 할 데이터의 기준 값 (criteria) 입니다. 이 값이 { } 이면 컬렉션의 모든 데이터를 제거합니다.|
|justOne|boolean|선택적(Optional) 매개변수이며 이 값이 true 면 1개 의 다큐먼트만 제거합니다. 이 매개변수가 생략되면 기본값은 false 로 서, criteria에 해당되는 모든 다큐먼트를 제거합니다.|

```
> db.books.find({"name": "Book1"})
{ "_id" : ObjectId("56c097f94d6b67aafdeb88ac"), "name" : "Book1", "author" : "Velopert" }
> db.books.remove({"name": "Book1"})
WriteResult({ "nRemoved" : 1 })
> db.books.find()
{ "_id" : ObjectId("56c08f3a4d6b67aafdeb88a3"), "name" : "MongoDB Guide", "author" : "Velopert" }
{ "_id" : ObjectId("56c08f474d6b67aafdeb88a4"), "name" : "NodeJS Guide", "author" : "Velopert" }
{ "_id" : ObjectId("56c097f94d6b67aafdeb88ad"), "name" : "Book2", "author" : "Velopert" }
```

***
# 강좌 3편 Document Query - find()
## 다큐먼트 쿼리
`db.COLLECTION_NAME.find(query, projection)`

|Parameter|Type|Description|
|---------|----|-----------|
|query|document	Optional(선택적)|다큐먼트를 조회할 때 기준을 정합니다. 기준이 없이 컬렉션에 있는 모든 다큐먼트를 조회 할때는 이 매개변수를 비우거나 비어있는 다큐먼트 { } 를 전달하세요.|
|projection|document Optional|다큐먼트를 조회할 때 보여질 field를 정합니다.|

__리턴값: cursor__  
criteria에 해당하는 다큐먼트를 선택하여 cursor를 반환한다. cursor는 쿼리 요청의 결과값을 가리키는 포인터이다. 이를 사용하여 데이터의 수를 제한할 수 있고 데이터를 정렬할 수 있다. 10분동안 사용하지 않으면 만료된다.

## 이쁘게 출력하기와 샘플데이터
`cursor.pretty()`
```
> db.articles.find().pretty()
{
        "_id" : ObjectId("56c0ab6c639be5292edab0c4"),
        "title" : "article01",
        "content" : "content01",
        "writer" : "Velopert",
        "likes" : 0,
        "comments" : [ ]
}
{
        "_id" : ObjectId("56c0ab6c639be5292edab0c5"),
        "title" : "article02",
        "content" : "content02",
        "writer" : "Alpha",
        "likes" : 23,
        "comments" : [
                {
                        "name" : "Bravo",
                        "message" : "Hey Man!"
                }
        ]
}
{
        "_id" : ObjectId("56c0ab6c639be5292edab0c6"),
        "title" : "article03",
        "content" : "content03",
        "writer" : "Bravo",
        "likes" : 40,
        "comments" : [
                {
                        "name" : "Charlie",
                        "message" : "Hey Man!"
                },
                {
                        "name" : "Delta",
                        "message" : "Hey Man!"
                }
        ]
}
```

## 쿼리 연산자
### 비교연산자
|Operator|Description|
|--------|-----------|
|$eq|(equals) 주어진 값과 일치하는 값|
|$gt|(greater than) 주어진 값보다 큰 값|
|$gte|(greather than or equals) 주어진 값보다 크거나 같은 값|
|$lt|(less than) 주어진 값보다 작은 값|
|$lte|(less than or equals) 주어진 값보다 작거나 같은 값|
|$ne|(not equal) 주어진 값과 일치하지 않는 값|
|$in|주어진 배열 안에 속하는 값|
|$nin|주어빈 배열 안에 속하지 않는 값|

10 < likes < 30 조회
```
> db.articles.find( { "likes": { $gt: 10, $lt: 30 } } ).pretty()
{
        "_id" : ObjectId("56c0ab6c639be5292edab0c5"),
        "title" : "article02",
        "content" : "content02",
        "writer" : "Alpha",
        "likes" : 23,
        "comments" : [
                {
                        "name" : "Bravo",
                        "message" : "Hey Man!"
                }
        ]
}
```

### 논리연산자
|Operator|Description|
|--------|-----------|
|$or|주어진 조건중 하나라도 true 일 때 true|
|$and|주어진 모든 조건이 true 일 때 true|
|$not|주어진 조건이 false 일 때 true|
|$nor|주어진 모든 조건이 false 일때 true|

 title 값이 article01이거나, writer값이 Alpha인 다큐먼트 조회
```
> db.articles.find({ $or: [ { "title": "article01" }, { "writer": "Alpha" } ] })
{ "_id" : ObjectId("56c0ab6c639be5292edab0c4"), "title" : "article01", "content" : "content01", "writer" : "Velopert", "likes" : 0, "comments" : [ ] }
{ "_id" : ObjectId("56c0ab6c639be5292edab0c5"), "title" : "article02", "content" : "content02", "writer" : "Alpha", "likes" : 23, "comments" : [ { "name" : "Bravo", "message" : "Hey Man!" } ] }
```
writer 값이 Velopert이고 likes 값이 10미만인 다큐먼트 조회
```
> db.articles.find( { $and: [ { "writer": "Velopert" }, { "likes": { $lt: 10 } } ] } )
{ "_id" : ObjectId("56c0ab6c639be5292edab0c4"), "title" : "article01", "content" : "content01", "writer" : "Velopert", "likes" : 0, "comments" : [ ] }

> db.articles.find( { "writer": "Velopert", "likes": { $lt: 10 } } )
{ "_id" : ObjectId("56c0ab6c639be5292edab0c4"), "title" : "article01", "content" : "content01", "writer" : "Velopert", "likes" : 0, "comments" : [ ] }
```

### regex 연산자
정규식으로 다큐먼트를 쿼리한다. 이 연산자는 아래와 같이 사용한다.
```
{ <field>: { $regex: /pattern/, $options: '<options>' } }
{ <field>: { $regex: 'pattern', $options: '<options>' } }
{ <field>: { $regex: /pattern/<options> } }
{ <field>: /pattern/<options> }
```
|Option|Description|
|------|-----------|
|i|대소문자 무시|
|m|정규식에서 anchor(^) 를 사용 할 때 값에 \n 이 있다면 무력화|
|x|정규식 안에있는 whitespace를 모두 무시|
|s|dot (.) 사용 할 떄 \n 을 포함해서 매치

title이 article0[1-2]인 다큐먼트 찾기
```
> db.articles.find( { "title" : /article0[1-2]/ } )
{ "_id" : ObjectId("56c0ab6c639be5292edab0c4"), "title" : "article01", "content" : "content01", "writer" : "Velopert", "likes" : 0, "comments" : [ ] }
{ "_id" : ObjectId("56c0ab6c639be5292edab0c5"), "title" : "article02", "content" : "content02", "writer" : "Alpha", "likes" : 23, "comments" : [ { "name" : "Bravo", "message" : "Hey Man!" } ] }
```

### where 연산자
javascript expression 사용할 수 있음

### elemMatch 연산자
엠베디드 다큐먼트('다큐먼트 in 다큐먼트'란 의미인 듯?) 배열을 쿼리하여 조건에 부합하는 다큐먼트들을 출력한다.  

comments 중에 Charlie가 작성한 덧글이 있는 다큐먼트 조회
```
> db.articles.find( { "comments": { $elemMatch: { "name": "Charlie" } } } )
{ "_id" : ObjectId("56c0ab6c639be5292edab0c6"), "title" : "article03", "content" : "content03", "writer" : "Bravo", "likes" : 40, "comments" : [ { "name" : "Charlie", "message" : "Hey Man!" }, { "name" : "Delta", "message" : "Hey Man!" } ] }
```

엠베디드 다큐먼트 배열이 아니라 name처럼 하나의 엠베디드 다큐먼트인 경우
```
{
    "username": "velopert",
    "name": { "first": "M.J.", "last": "K."},
    "language": ["korean", "english", "chinese"]
}
```

다음과 같이 쿼리
```
> db.users.find({ "name.first": "M.J."})
```

엠베디드 다큐먼트 배열이 아닌 일반 배열인 경우
```
> db.users.find({ "language": "korean"})
```

## Projection - find()의 두번째 인자
쿼리값에서 보여질 필드를 정한다.

```
> db.articles.find( { } , { "_id": false, "title": true, "content": true } )
{ "title" : "article01", "content" : "content01" }
{ "title" : "article02", "content" : "content02" }
{ "title" : "article03", "content" : "content03" }
```

### slice 연산자
출력 개수의 limit를 설정한다.  

title이 article03인 다큐먼트에서 덧글이 하나만 보이게 출력
```
> db.articles.find({"title": "article03"}, {comments: {$slice: 1}}).pretty()
{
        "_id" : ObjectId("56c0ab6c639be5292edab0c6"),
        "title" : "article03",
        "content" : "content03",
        "writer" : "Bravo",
        "likes" : 40,
        "comments" : [
                {
                        "name" : "Charlie",
                        "message" : "Hey Man!"
                }
        ]
}
```

### elemMatch 연산자
엠베디드 다큐먼트 배열의 특정 필드만 projection하여 보여준다.

위의 쿼리연산자-`elemMatch` 연산자 예제에서 commands중 Charlie가 작성한 덧글이 있는 다큐먼트를 조회했을 때, 게시물의 제목과 Charlie의 덧글 부분만 읽고 싶을 때는 어떻게 하는가?

```
db.articles.find(
    {
        "comments": {
            $elemMatch: { "name": "Charlie" }
        }
    },
    {
        "title": true,
        "comments.name": true,
        "comments.message": true
    }
)
```
의도와는 다르게 Delta의 덧글도 출력된다.
```
{
        "_id" : ObjectId("56c0ab6c639be5292edab0c6"),
        "title" : "article03",
        "comments" : [
                {
                        "name" : "Charlie",
                        "message" : "Hey Man!"
                },
                {
                        "name" : "Delta",
                        "message" : "Hey Man!"
                }
        ]
}
```
따라서 `elemMatch`연산자를 projection연산자로 사용하면 이를 구현할 수 있음
```
db.articles.find(
     {
         "comments": {
             $elemMatch: { "name": "Charlie" }
         }
     },
     {
         "title": true,
         "comments": {
             $elemMatch: { "name": "Charlie" }
         },
         "comments.name": true,
         "comments.message": true
     }
)
{ "_id" : ObjectId("56c0ab6c639be5292edab0c6"), "title" : "article03", "comments" : [ { "name" : "Charlie", "message" : "Hey Man!" } ] }
```

***
# 강좌 4편 find method의 활용 - sort(), limit(), skip()
cursor 객체가 가진 기능(정렬, 출력물 개수 제한)을 활용해보자.

## 샘플 데이터
```
[
    { "_id": 1, "item": { "category": "cake", "type": "chiffon" }, "amount": 10 },
    { "_id": 2, "item": { "category": "cookies", "type": "chocolate chip" }, "amount": 50 },
    { "_id": 3, "item": { "category": "cookies", "type": "chocolate chip" }, "amount": 15 },
    { "_id": 4, "item": { "category": "cake", "type": "lemon" }, "amount": 30 },
    { "_id": 5, "item": { "category": "cake", "type": "carrot" }, "amount": 20 },
    { "_id": 6, "item": { "category": "brownies", "type": "blondie" }, "amount": 10 }
]
```

## 정렬 - sort()
`cursor.sort(DOCUMENT)`  
데이터를 정렬 할 때 사용한다. 매개변수로 어떤 키를 사용할지를 아래와 같은 구조의 다큐먼트로 전달한다.  
```
{ KEY: value }
```
KEY는 filed의 이름, value는 1(오름차순) 또는 -1(내림차순)  
또한 여러 키를 입력할 수 있고 먼저 입력한 키가 우선권을 갖는다.  

id의 값을 사용하여 오름차순으로 정렬하기
```
> db.orders.find().sort( { "_id": 1 } )
{ "_id" : 1, "item" : { "category" : "cake", "type" : "chiffon" }, "amount" : 10 }
{ "_id" : 2, "item" : { "category" : "cookies", "type" : "chocolate chip" }, "amount" : 50 }
{ "_id" : 3, "item" : { "category" : "cookies", "type" : "chocolate chip" }, "amount" : 15 }
{ "_id" : 4, "item" : { "category" : "cake", "type" : "lemon" }, "amount" : 30 }
{ "_id" : 5, "item" : { "category" : "cake", "type" : "carrot" }, "amount" : 20 }
{ "_id" : 6, "item" : { "category" : "brownies", "type" : "blondie" }, "amount" : 10 }
```

amount 값으로 오름차순 정렬하고, 정렬한 값에서 id값은 내림차순으로 정렬하기
```
> db.orders.find().sort( { "amount": 1, "_id": -1 } )
{ "_id" : 6, "item" : { "category" : "brownies", "type" : "blondie" }, "amount" : 10 }
{ "_id" : 1, "item" : { "category" : "cake", "type" : "chiffon" }, "amount" : 10 }
{ "_id" : 3, "item" : { "category" : "cookies", "type" : "chocolate chip" }, "amount" : 15 }
{ "_id" : 5, "item" : { "category" : "cake", "type" : "carrot" }, "amount" : 20 }
{ "_id" : 4, "item" : { "category" : "cake", "type" : "lemon" }, "amount" : 30 }
{ "_id" : 2, "item" : { "category" : "cookies", "type" : "chocolate chip" }, "amount" : 50 }
```

## 출력 개수 제한 - limit()
`cursor.limit(value)`  
출력할 데이터의 개수를 제한할 때 사용된다. value는 제한할 개수

## 출력의 시작부분 설정 - skip()
`cursor.skip(value)`  
출력할 데이터의 시작부분을 설정한다. value값 개수의 데이터를 생략하고 그 다음부터 출력한다.  

2개의 데이터를 생략하고 그 다음부터 출력하기
```
> db.orders.find().skip(2)
{ "_id" : 3, "item" : { "category" : "cookies", "type" : "chocolate chip" }, "amount" : 15 }
{ "_id" : 4, "item" : { "category" : "cake", "type" : "lemon" }, "amount" : 30 }
{ "_id" : 5, "item" : { "category" : "cake", "type" : "carrot" }, "amount" : 20 }
{ "_id" : 6, "item" : { "category" : "brownies", "type" : "blondie" }, "amount" : 10 }
```

## 응용
`sort()`, `limit()`, `skip()` 역시 리턴값은 cursor이기에 위 메서드를 중첩하여 사용가능하다.

```
> var showPage = function(page){
     return db.orders.find().sort( { "_id": -1 } ).skip((page-1)*2).limit(2);
     }
> showPage(1)
{ "_id" : 6, "item" : { "category" : "brownies", "type" : "blondie" }, "amount" : 10 }
{ "_id" : 5, "item" : { "category" : "cake", "type" : "carrot" }, "amount" : 20 }
> showPage(2)
{ "_id" : 4, "item" : { "category" : "cake", "type" : "lemon" }, "amount" : 30 }
{ "_id" : 3, "item" : { "category" : "cookies", "type" : "chocolate chip" }, "amount" : 15 }
> showPage(3)
{ "_id" : 2, "item" : { "category" : "cookies", "type" : "chocolate chip" }, "amount" : 50 }
{ "_id" : 1, "item" : { "category" : "cake", "type" : "chiffon" }, "amount" : 10 }
```

***
# 강좌 5편 Document Update - update()
## 정의
몽고디비에서는 `update()`메서드를 통해 데이터를 수정할 수 있고 다음과 같은 구조를 지닌다.
```
db.collection.update(
    <query>,
    <update>,
    {
        upsert: <boolean>,
        multi: <boolean>,
        writeConcern: <document>
    }
)
```
collection안에 다큐먼트를 수정한다. 이 메서드를 사용하여 특정 필드를 수정하거나 이미 존재하는 다큐먼트를 교체할 수 있다.  

|Parameter|Type|Description|
|---------|----|-----------|
|query|document|업데이트 할 document의 criteria 를 정합니다. find() 메소드 에서 사용하는 query 와 같습니다.|
|update|document|document에 적용할 변동사항입니다.|
|upsert|boolean	Optional (기본값: false)|이 값이 true 로 설정되면 query한 document가 없을 경우, 새로운 document를 추가합니다.|
|multi|boolean	Optional (기본값: false)|이 값이 true 로 설정되면, 여러개의 document 를 수정합니다.|
|writeConcern|document	Optional|wtimeout 등 document 업데이트 할 때 필요한 설정값입니다. 기본 writeConcern을 사용하려면 이 파라미터를 생략하세요. 자세한 내용은 [매뉴얼](https://docs.mongodb.com/v3.2/reference/write-concern/)을 참조해주세요.|

## 활용
### 샘플 데이터
```
db people insert( [
    { name: "Abet", age: 19 },
    { name: "Betty", age: 20 },
    { name: "Charlie", age: 23, skills: [ "mongodb", "nodejs"] },
    { name: "David", age: 23, score: 20 }
])
```

### 특정 필드 수정하기
특정 필드값을 수정하고자 할 경우 `$set` 연산자를 사용한다.
```
// Abet document 의 age를 20으로 변경한다
> db.people.update( { name: "Abet" }, { $set: { age: 20 } } )
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

### 다큐먼트 교체하기
교체(replace)될 때, id는 변경 안함
```
// Betty document를 새로운 document로 대체한다.
> db.people.update( { name: "Betty" }, { "name": "Betty 2nd", age: 1 })
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

### 특정 필드 제거하기
```
// upsert 옵션을 설정하여 Elly document가 존재하지 않으면 새로 추가
> db.people.update( { name: "Elly" }, { name: "Elly", age: 17 }, { upsert: true } )
WriteResult({
    "nMatched" : 0,
    "nUpserted" : 1,
    "nModified" : 0,
    "_id" : ObjectId("56c893ffc694e4e7c8594240")
})
```

### 배열에 값 추가하기
`$push` 연산자를 사용
```
// Charlie document의 skills 배열에 "angularjs" 추가
> db.people.update(
    { name: "Charlie" },
    { $push: { skills: "angularjs" } } )
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```
### 배열값 여러개 추가 + 오름차순 정렬하기
```
// Charlie document의 skills에 "c++" 와 "java" 를 추가하고 알파벳순으로 정렬
> db.people.update(
    { name: "Charlie" },
    { $push: {
        skills: {
            $each: [ "c++", "java" ],
            $sort: 1
        }
    }
})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

### 배열에 값 제거하기
`$pull` 연산자를 사용
```
// Charlie document에서 skills 값의 mongodb 제거
> db.people.update(
    { name: "Charlie" },
    { $pull: { skills: "mongodb" }
})
```

### 배열에서 값 여러개 제거하기
`$in` 연산자를 사용한다.
```
// Charlie document에서 skills 배열 중 "angularjs" 와 "java" 제거
> db.people.update(
    { name: "Charlie" },
    { $pull: { skills: { $in: ["angularjs", "java" ] } } }
    )
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

### 그 외에 ...
[메뉴얼 참조](https://docs.mongodb.com/v3.2/reference/operator/update/)

***
# 강좌 6편 Index
데이터 __쿼리__ 를 효율적으로 해줄 수 있다.  
몽고디비의 필드에 인덱스를 걸면 다큐먼트의 포인터값으로 이루어진 B-Tree(Balanced Binary Search Tree)를 만든다.
## 인덱스의 종류
### 기본 인덱스 `_id`
모든 몽고디비의 컬렉션은 `_id`필드에 인덱스가 존재하며, 따로 지정하지 않더라도 자동으로 ObjectId로 설정해 준다. 이 인덱스는 __유일__ 하고 몽고디비 클라이언트가 같은 `_id`를 가진 문서를 중복적으로 추가하는 것을 방지한다.

### 단일(Single) 필드 인덱스
사용자가 지정할 수 있는 인덱스

### 복합(Compound) 필드 인덱스
두개 이상의 필드를 사용하는 인덱스

### 멀티키(Multikey) 인덱스
배열인 필드를 인덱스로 사용한다.

### 공간적(Geospatial) 인덱스
지도의 좌표와 같은 데이터를 효율적으로 쿼리하기 위한 인덱스
(참고)[https://docs.mongodb.org/manual/core/geospatial-indexes/]

### 텍스트 인덱스
텍스트 데이터를 호율적으로 쿼리하기 위한 인덱스
(참고)[https://docs.mongodb.org/manual/core/index-text/]

### 해시 인덱스
B-Tree 대신 해시 자료구조를 사용한다. 해시의 검색 효율은 B-Tree보다 좋지만, 정렬을 하지 않는다.


## 인덱스 생성하기
인덱스를 생성할 때는 `createIndex()` 메서드를 사용한다. 인수로 아래와 같은 다큐먼트를 전달한다. 키는 필드이고 값은 1(오름차순) or -1(내림차순)이다.
```
db.COLLECTION.createIndex( { KEY: value } )
```

## 인덱스 속성
인덱스에 속성을 추가할 경우 두번쨰 인자로 아래와 같은 다큐먼트를 전달한다.
```
db.COLLECTION.createIndex( { KEY: 1 }, {  PROPERTY: true } )
```
인덱스에 적용할 수 있는 속성은 4가지가 있다.
### Qnique 속성
`_id`처럼 유일값만 존재할 수 있는 속성이다.
```
db.userinfo.createIndex( { email: 1 }, { unique: true } )
```

복합 인덱스에서도 적용할 수 있다.
```
db.userinfo.createIndex( { firstName: 1, lastName: 1 }, { unique: true } )
```

### Partial 속성
다큐먼트의 조건을 정하여 일부의 다큐먼트에만 인덱스를 적용할 떄 사용한다. 이를 활용하면 필요한 부분에만 인덱싱하여 저장공간을 아끼고 속도를 더 높일 수 있다.
```
// visitor 값이 1000보다 높은 document만 name필드에 인덱스를 적용한다.
db.store.createIndex(
   { name: 1 },
   { partialFilterExpression: { visitors: { $gt: 1000 } } }
)
```

### TLL 속성
Date 또는 Date 배열의 필드에서 적용할 수 있는 속성이다. 이 속성을 사용하면 특정시간이 만료된(expired) document를 컬렉션에서 제거할 수 있다.

```
// notifiedDate가 현재 시간과 1시간 이상 차이나면 제거
db.notifications.createIndex( { "notifiedDate": 1 }, { expireAfterSeconds: 3600 } )
```
다큐먼트가 만료되어 제거될 때는 시간이 정확하지 않다.(60초마다 검사)

## 인덱스 조회
`db.COLLECTION.getIndexes()`

## 인덱스 제거
`db.COLLECTION.dropIndex( { KEY: 1 } )`

모든 인덱스를 제거할 때는 `dropIndexes()`를 사용한다.
