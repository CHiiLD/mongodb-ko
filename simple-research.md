# 몽고디비의 간단한 리서치 내용 정리
## 구조
- mongod는 실제 데이타 베이스 핸들링 프로세스로 mysqld와 유사
- 앞단에 mongos 라는 프로세스를 띄워서 클러스터 구성을 하면, mongos가 로드 밸런서 역할을 함

## 클러스터링을 할경우
- Sharding을 사용하여 데이타를 분산 저장해야 함
- 이 경우 같은 __shard내에 mongod를 3 copy로 replication__ 하여 데이타 유실을 방지를 권고한다.
- 고급 문서 대부분 내용이 Shard 구성과 Index 구성이다. 이게 키 포인트인듯
※ 이 과정은 Redundant한 하드웨어 구성으로 인하여 하드웨어 코스트를 올릴 수 있다.

## 성능 부분에서는
- mongodb는 memory 기반의 index를 사용하여 cassandra나 hbase 에 비하면 높은 read performance를 보장한다.
- 단 메모리가 충분해야 한다는 이야기고 반대로 이야기 하면 비싸고 확장성에 제약이 있다는 이야기다.

## 용량 부분에서
- 사례 문서를 보니까는 1 node에 3TB 저장 가능한 문서가 있었고
- 궁극적으로 1000 node를 클러스터로 묶을 수 있다고 했다. 즉 peta byte급의 데이타 저장이 가능하다는 것이다. (실제로 가능할지는 모르겠다.)
- 여기에 아울러 12billion record (120억 레코드)
데이타 상으로만 보면 꽤 큰 데이타 저장이 가능한데, 대부분 문서는 중소형에서만 mongodb를 권장하고 있다.

## 저장 레코드
- GridFS 기반으로 파일을 저장할 수 있다.
- 대용량 파일은 Chunk 단위로 나눠서 저장이 가능하다. (자동으로 나눠줌)

## 기술지원
- 10gen에서 컨설팅과 Support 서비스 가능
- foursquare 레퍼런스가 있음. (나름 레퍼런스가 있다.)

## 결론
- __중소형__ 에서만 사용하는 것이 권장됨
- 아키텍쳐 디자인에서는 __Sharding과 Index__ 구성이 관건
- 하드웨어 아키텍쳐 구조에서는 메모리를 충분히 배치할것
- 그리고 결코 __싸지 않다.__ (3 Copy)

***
출처: [조대엽의 블로그](http://bcho.tistory.com/604)
