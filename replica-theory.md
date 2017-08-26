# 몽고디비의 복제
## 복제 시스템
### Master-Slave
MongoDB의 복제 정책은 기본적으로 __Master-Slave__ 방식을 채택하고 있으며, master가 죽더라도 slave들 중에서 master를 선출할 수 있는 master 선출 방법을 채택하고 있다. 즉, Master가 죽으면, slave들이 투표를 진행하고 투표된 결과에 따라 새로운 master가 선출된다.

![그림 2-1](./images/pic2-1.png)

[그림 2-1]은 MongoDB의 복제 정책을 보여준다. MongoDB의 __master는 쓰기 연산__ 을 담당한다. 즉, 일반 Master-Slave 방식과 동일하게 쓰기는 master에서만 이루어진다. 이때 MongoDB는 쓰기 연산을 데이터 저장소와 Oplog라는 두 군데 영역에 저장한다. 데이터 저장소에는 B+ 트리로 구성된 데이터 저장소를 말하는 것으로, 쓰기 연산을 수행한 결과를 저장한다. 반면 __Oplog는 데이터 저장소에 저장된 데이터와는 달리 연산 수행과 관련된 명령 자체를 타임스탬프와 같이 저장한다__.  

### oplog
MongoDB의 slave는 아주 빠른 주기적으로 master에게 자신의 optime보다 큰 oplog를 달라고 요청한다. Slave의 요청은 master에 oplog 데이터를 요청할 때, 질의 요청 옵션을 QueryOption_AwaitData으로 보낸다. QueryOption_AwaitData는 대기하는 일정 시간 안에 응답할 데이터가 존재한다면 바로 응답하고, 응답할 데이터가 없다면, 일정 시간을 대기한다는 것을 의미한다. Oplog 질의에 대한 대기 시간은 5초이다. 즉, 5초안에 master에서 쓰기 연산이 발생하면 바로 데이터를 응답해 주고, 5초안에 쓰기 연산이 발생하지 않는다면, 데이터가 존재하지 않는다는 응답을 보내준다. Slave는 요구한 Oplog의 데이터가 존재하면 자신의 Oplog에 데이터를 저장한 다음에 바로 mater에 다시 Oplog 질의를 수행한다.  

### 동기화 처리
Slave의 동기화 처리는 쓰레드 한 개가 지속적으로 담당하고 있다. 만약, master와의 연결이 단절되어 Oplog를 동기화 시키지 못하게 되는 경우는 자신의 Oplog의 마지막 연산 시간을 저장하고, Oplog를 비우고 난 다음에 메모리에 보관된 모든 데이터를 데이터 저장소에 저장시킨다. 그리고, 5초 이후에 다시 master와의 연결을 시도한다.  


Master 역시 slave와의 동기화를 위해 한 개의 쓰레드를 만들어 slave와의 통신을 담당하고 slave가 요구하는 데이터를 전달한다. 따라서 master는 능동적 동기화 보다는 수동적 동기화 입장이므로, slave들의 요구에 따라 데이터를 전송하는 부담만 지게 된다. 이러한 MongoDB의 철학은 master의 역할을 쓰기 연산에 집중 하도록 구성하여 빠른 속도를 구사할 수 있도록 만들고 있다. 실제 성능 테스트를 수행하였을 경우에도, 복제를 설정한 상태에서 master의 부하가 가중되지는 않는다. 성능 이슈는 5% 정도로 복제가 전체 시스템의 성능 여하를 결정하지 않는다. 다만, 복제를 위한 slave의 개수를 많이 둘 경우에는 master의 동기화 요구가 많아지게 되므로, 그 만큼의 속도 저하는 고려하여야 한다.

## 복제 동기화와 마스터 선출
### heartbeat

![그림 2-2](./images/pic2-2.png)

[그림 2-2]의 MongoDB는 한 개의 master와 두 개의 slave로 구성된 복제 집합replica set을[1]구성하고 있다. MongoDB는 복제 집합으로 구성된 각각의 노드는 자신을 제외한 다른 노드들이 죽었는지 살았는지를 검사하기 위해 [그림 2-2]과 같은 __heartbeat__ 를 사용한다. MongoDB의 heartbeat는 2초 단위로 수행되며, heartbeat을 받은 서버는 자신의 상태 코드를 heartbeat을 요청한 서버에 보내준다. [표 2]는 heartbeat을 통해 전달된 서버 상태 코드를 보여준다.

|상태 코드|	내용|
|-------|---|
|RS_STARTUP|	서버가 시작 중이거나 또는 복제 집합의 초기화를 수행하는 상태|
|RS_STARTUP2|	초기화를 위한 복제 환경 로드는 완료하였지만, primary[2]를   선출하는 상태|
|RS_PRIMARY|	서버 자신이 primary 상태|
|RS_SECONDARY|	서버 자신이 secondary[3] 상태|
|RS_RECOVERING|	서버가 secondary로 상태가 변경되기 전에 primary로부터 복제 동기화를 수행하는 상태
|RS_ROLLBACK|	서버가 secondary로 상태가 변경되기 전에 primary로부터 복제 동기화를 수행하는 상태로,  서버가 primary보다 더 많은 데이터를 가지고 있어서 primary 상태로 저장된 데이터로 되돌리는 상태|
|RS_FATAL|	서버가 복제 집합 안에서 네트워크 단절과 같이 완전한 offline 상태는 아니지만, 심각한 문제가 발생한 상태|
|RS_SHUNNED|	서버가 어떠한 복제 집합에도 속하지 않은 상태|

### 마스터 선출
Master 서버의 heartbeat는 항상 복제 집합을 구성하고 있는 노드 개수의 과반수만큼을 유지하고 있어야 한다. 만약 master 서버가 과반수의 heartbeat을 가지고 있지 않다면, 서버는 slave로 변환되며, 복제 집합은 master 부재에 따른 투표를 시행한다. MongoDB의 master 선출과정은 다른 Google의 PAXSOS와 유사하다. 단지 차이점이 있다면, master가 될 수 있는 자격 조건에 대한 옵션 priority와 votes를 가지고 있다는 차이점이 있다. __Priority는 자신이 master가 될 수 있는 우선 순위를 나타내는 것으로 priority가 높은 서버가 master가 될 가능성이 높다.__ 만약 priority가 0이라면 이는 자신 자신은 master로 설정할 수 없음을 나타낸다. 또한 __votes는 투표할 수 있는 개수를 의미하는 것으로, master는 자신을 포함한 복제 집합의 노드 개수의 과반수 투표를 가져야 한다.__ 그렇다면, votes와 priority와의 차이점은 무엇인가? Votes는 투표권을 말한다. 즉, 한 복제 집합에서 보유한 투표권은 각각의 노드가 가지고 있는 투표권을 모두 더한 것을 말한다. 아무런 값도 설정하지 않았다면 투표권은 한 노드 당 한 개의 값을 가질 것이다. 예를 들어 보자. 총 4개의 노드로 구성된 복제 집합이 [그림 2-3]과 같이 구성되었다고 가정하자.

![그림 2-3](./images/pic2-3.png)


[그림 2-3]의 (a)는 정상적인 상태의 복제 집합으로 총 투표권은 6개로 설정되어 있다. 그런데, (b)번과 같이 가장 많은 투표권을 가지고 있는 노드가 죽었다면, 갑자기 3개의 투표권이 없어진다. 그러면 [그림 2-3]의 복제 집합은 기존에 존재하던 6개의 투표권 중에서 살아있는 다른 노드들의 투표권을 더해도 과반수를 넘지 못한다. 따라서, (b)와 같은 시스템에서는 master를 선출할 수 없다. MongoDB의 votes가 이해되었다면, priority와의 차이점도 정확한 의미를 알 수 있다. Votes는 가장 중요한 한 개의 노드를 기준으로 해당 노드가 죽었을 경우에 전체 시스템을 read-only로 만들 수 있고, priority는 master의 가중치를 두어서, 언제든지 master 선출과정에서 master가 될 확률을 높인다는 것이다. 두 개가 같은 의미지만, 약간 다른 의미를 가지고 있음을 주의하자.

#### 마스터의 자격 조건
+ 복제 집합에서 과반수 노드들과 연결을 유지하는 있어야 한다.
+ Priority가 0보다 커야 한다.
+ 서버가 유지하고 있는 optime이 연결될 수 있는 다른 노드들의 optime 중에서 가장 최신 opime을 기준으로 10초 이내에 있어야 한다.

#### 투표 과정
1. 자신의 master가 될 수 있는 서버인지 판단하고, master가 될 수 있는 서버는 다음과 같은 일을 수행한다.
1. 자신이 master임을 인지하고 아주 짧은 시간 동안 기다린다.
1. 자신이 master임을 다른 노드에 통보한다.
1. 만약, 다른 노드들로부터 전달받은 메시지가 있다면, 자신의 priority와 비교하여 낮거나 또는 자신의 optime 보다 최신의 상태가 아니면 거부권(NO)을 발동한다.
1. 만약, 거부권을 행사했다면, 자신의 master가 될 수 있다는 것이므로, 거부권을 행사한 서버는 YES를 과반수 이상 받아야만 한다.
1. 만약, 2)번에서 거부권을 행상하지 않은 서버는 자신이 master의 자격이 없다는 것을 의미하는 것으로, 1)번에서 발송한 메시지에 대해 1분안에 거부권을 모두 받아야만 하며, 거부권을 모두 수신한 서버는 투표를 완료하게 된다.[6]


위의 절차에서 4)번의 1분은 최대 대기 시간을 의미하는 것으로, 대부분 네트워크 부하 또는 서버의 부하가 발생하지 않는다면, 10초안에 투표는 완료되어 master가 선출된다. 초기 시스템이 로딩하였을 때의 master 선출 시간은 약 5초 정도이다. MongoDB의 master 선출 과정에서 priority에 의해 가장 최신의 업데이트 정보를 가지고 있지 않은 서버가 master로 선출될 수 있다. 만약 이러한 경우에 __복제 동기화 과정에서 자신의 optime과 master의 optime을 비교하여 master보다 최신 데이터는 버리게 된다.__ 이와 같이 버려진 데이터는 나중에 관리자를 통해 __수동 복구할 수 있도록 BSON 형태의 파일로 저장__ 된다. 이와 같은 경우는 master 선출을 강제로 특정 서버로 유지하기 위해 사용될 경우에 발생되며, master 후보군에서 보여지듯이 10초 이내의 데이터만 유실 가능성이 발생된다.

## 복제의 한계
>한 복제 집합을 구성할 수 있는 노드의 최대 개수는 12개이다.  
>한 복제 집합에서 투표할 수 있는 노드의 최대 개수는 7개이다.

상기와 노드의 개수에 제한을 둔 것은 성능과 관련이 있다. 즉 Slave 개수의 제한은 master의 부하를 결정하고 투표 노드의 제한은 master 선출의 최대 시간과 관련 있다. 아마 10gen이 실험적으로 위와 같은 한계사항을 결정한 것이 아닌가 판단된다.

## 저널링 시스템
MongoDB는 데이터 손실을 최소화하기 위한 방법으로 저널링을 제공한다. 저널링이란 일종의 로그와 같은 것으로 MongoDB에 __데이터의 변화에 따른 모든 연산에 대해 로그를 적재__ 한다. 저널링 시스템은 [그림 2-4]와 같은 구조로 구성된다.

![그림 2-4](./images/pic2-4.png)

[그림 2-4]의 (a)는 일반 데이터에 대한 저널링을 의미하고, (b)는 인덱스 데이터에 대한 저널링을 의미한다. Oplog는 데이터 연산 중에 쓰기, 수정, 삭제와 같이 데이터 변동이 발생될 때의 연산 자체를 저장하여 master와 slave간의 동기화를 수행하는 high-level transaction을 보장하는 저장소인 반면, 저널링은 mongod 한 노드에 개별적으로 설정하여 저장되는 low-level 로그와 같은 구성이다.

MongoDB는 __충돌에 의해, 복구를 수행하고자 한다면, 우선 저널링을 통해 개별 mongod들의 복구를 수행한다. 저널 파일에 저장된 데이터는 [그림 2-4]와 같이 일반 데이터 저장소의 데이터와 Oplog의 데이터를 모두 저장하고 있어서, 일반 데이터 저장소만 복구하는 기능 외에 복제를 위한 Oplog의 동기화 데이터까지 복구가 가능하다.__

데이터 중복성에 대해서 논의해 보자. [그림 2-4] (a)와 같이 복제와 저널링을 모두 사용할 경우에는 한 개의 데이터에 대해서 총 4X의 중복성을 가지고, (b)와 같은 인덱스에서는 2X의 중복성을 가진다. 데이터와 인덱스의 사용비율에 대한 일반적 통계 비율에 의하면 데이터 중복성은 총 2.5X 정도 사용된다.

저널링은 데이터 중복성을 가지기 때문에, 시스템 전체 속도와도 관련 있다. 아무래도 저널링을 사용하면 속도는 떨어진다. 이러한 문제를 위해 MongoDB는 group commits를 제공한다. Group commits란 저널링 로그가 발생할 때 마다, 한번씩 데이터를 파일에 저장하는 것이 아니라, 일정 시간 동안 발생된 데이터를 한 번에 저장하는 방식을 말한다. 디폴트 값은 100ms 로 설정되어 있으며, 관리자에 의해 기동할 때 2 ~ 300ms의 범위의 값을 설정할 수 있다.

## 데이터의 유실 가능성

앞 절에서 살펴보았듯이 MongoDB의 데이터 유실 가능성이 존재하지 않는 것은 아니다. 복제의 경우는 slave가 아무리 빨리 데이터 동기화를 한다고 하여도, master와의 통신 지연 시간만큼의 차이를 가질 수 있고, 저널링 역시 group commits의 시간에 의해 차이가 발생된다.

하지만, slave의 Oplog 동기화는 시스템 부하가 없는 경우라면, 쓰레드에 의해 아주 빠르게 주기적으로 동기화를 수행하기 때문에, 쓰기 연산이 엄청난 부하를 가지는 않는 상태에서는 Oplog의 동기화로 문제시 되지 않는다. 다만, 부하를 견디지 못해 서버가 죽는 경우라면, Oplog에 동기화 되지 않은 채 남아있는 데이터 연산을 잃어버리는 현상이 나타난다.

저널링은 데이터 저장소에 문제가 발생하여 데이터가 유실되었을 경우에, 복구를 수행하는 것이다. 복구는 MongoDB를 재 기동시킬 때 또는 복구 명령을 직접 수행하였을 경우에 이루어진다. __저널링__ 파일에 데이터를 저장하는 방법 역시, 앞 절에서 살펴보았듯이 디폴트 값이 100ms의 오차를 가지고 있다. __만약 100ms 안에 데이터가 존재하여 시스템이 죽어서 메모리에 저장된 데이터가 날라갔다면, 이는 복구가 불가능하다.__

이러한 경우는, 복제로 설정된 상태에서 master와의 Oplog를 통한 동기화를 수행할 수 있으므로, 일정 부분 복구가 가능하다. 하지만, master 자체가 죽었다면 복구가 불가능한 데이터가 일부 남아 있을 수 있다.(Oplog의 동기화는 slave가 master에 요청하는 것이기 때문에, slave는 master의 데이터보다 같거나 적은 데이터를 유지한다.) MongoDB는 이러한 문제점을 해결하기 위해 일관성 정책에서 복제의 Oplog와 관련된 REPLICAS_SAFE, MAJORITY를 제공하고, 저널링과 관련된 JOURNAL_SAFE를 제공한다. __REPLICAS_SAFE__ 는 master를 제외한 적어도 한 개의 slave의 Oplog가 동기화가 완료된 상태를 의미하고, __MAJORITY__ 는 임의의 복제 집합으로 구성된 노드들의 과반수가 Oplog 동기화에 성공하였음을 의미한다. 예를 들어, 복제 레벨을 5로 구성하였다면(Master 1대와 slave 4대로 구성), master를 포함한 3대가 동기화 되었음을 알려준다. __JOURNAL_SAFE__ 는 저널 파일에 로그가 저장될 때까지 쓰기 연산을 기다리는 것으로, 만약 group commits가 100ms로 설정되어 있다면, 한 번의 쓰기 연산은 최대 100ms를 기다릴 수 있음을 나타낸다.

이외에도 __FSYNC_SAFE__ 모드가 있는데, 이는 MongoDB가 메모리에 있는 데이터를 물리적 저장소에 저장시킬 때까지 쓰기 연산을 기다리는 것을 의미한다. MongoDB의 fsync는 디폴트 값이 60초로 설정되어 있다. 따라서 한 번의 쓰기 연산은 최대 60초를 기다릴 수 있다.

MongoDB의 데이터 유실과 관련된 사항은 일반 데이터베이스에서 고려될 수 있는 일반적인 사항이다. MongoDB 역시 이러한 부분을 옵션을 이용하여 처리할 수 있도록 제공하고 있고, 또는 쓰기 연산에 WriteConcern을 조정하여 특정 연산에 대한 트랜잭션을 보장할 수 있도록 API를 제공하고 있다.

>일관성과 데이터 유실 문제가 심각한 트랜잭션 보장 데이터에 대해서는 복제로 구성된 시스템에서는 REPLICAS_SAFE로 처리하는 것이 바람직하며, standalone으로 사용하는 시스템에서는 저널링의 group commits 시간을 줄이고, JOURNAL_SAFE를 사용하는 것이 바람직하다. 물론 이러한 두 가지 방법 모두 성능과 밀접한 관련이 있으므로, 프로그래머에 의한 적절한 조절이 필요하다.

>마지막으로, 복제가 이유 없이 수행되지 않는다는 것은, 첫째 시스템 부하가 너무 많아서 slave의 동작이 의심스러운 경우, 또는 master의 fail에 대해서 master 선출과정에서 slave들이 master 정보를 갱신하지 못해, Oplog의 동기화를 수행하지 못하는 경우이다.

>전자의 경우는 한 노드에 여러 개의 slave를 설정하는 경우이거나 다른 프로세스의 동작이 MongoDB의 slave가 원활하게 동작할 수 없도록 막는 경우이다. 가급적 MongoDB의 coordinate하는 과정에서 slave를 중복하여 설치하는 것은 복제 동기화에 부담을 줄 수 있으므로, 피하는 것이 바람직하다. 후자의 경우는 예전 버전에서 이러한 문제점들이 발견되어 master를 읽는 경우가 있었으나, 현재는 그러한 버그는 모두 수정된 것으로 알고 있다. 만약 slave를 arbiter와 같이 사용할 경우에, arbiter는 master 선출을 위한 투표 이외에 아무런 작업을 하지 않으므로, 복제와는 관련이 없다.

>완벽한 프로그램은 존재하지 않는다. 어떤 이유에서라도 프로그램 오동작으로 인해, 프로세스가 충돌이 날 수 있다. 이러한 경우를 대비하기 위해, 중요한 시스템에서는 복제 레벨을 올려두거나 또는 일관성 정책을 조절하는 등 여러 가지 대비하여 시스템을 구축하는 것이 가장 좋은 fail-over 방법이다.

# 몽고디비의 락
C++로 개발된 MongoDB는 boost의 shared_mutex를 이용하여 락 시스템을 개발하였다. shared_mutex는 기본적으로 one writer many reader의 구조를 가지기 때문에 __MongoDB는 쓰기 락이 한 개만 존재__ 한다. 이러한 점은 MongoDB의 쓰기 연산에서 부하가 발생할 경우, 전체적인 시스템 부하를 야기할 수 있다.  하지만, 10gen에서는 쓰기 연산 속도가 매우 빠르기 때문에, MongoDB의 락이 부하를 발생시키지 않는다고 말한다. 사실 NoSQL은 트랜잭션이 없기 때문에 쓰기 연산에 대한 부하가 RDB 처럼 심하지 않다. 즉 __근본적인 쓰기만 수행하면 되기 때문에 매우 빠른 특징을 가진다. 또한 MongoDB는 1차적으로 메모리에 데이터를 쓰기 때문에 쓰기 연산 속도가 빠르다는 것이 보장된다.__

그렇다고, MongoDB의 락 시스템이 완전하다고 볼 수 없다. MongoDB는 쓰기 락을 시스템에 한 개를 유지하기 때문에, 아무래도 최대 쓰기 연산의 한계를 가진다. 이말은 클라이언트의 개수와는 상관 없이 최대 저장 속도가 락에 의해서 결정된다는 것이다. 이러한 문제점은 버전 2.2에서 컬랙션 별로 쓰기 락을 구현하여 어느정도 해소시켰다. 본 장에서는 이러한 MongoDB의 락 시스템에 대한 구조를 살펴보도록 한다.

## Boost의 shared_mutex
Boost에서는 일반 mutex의 기본 개념보다 좀 더 유연한 형태의 락을 설정할 수 있도록 __upgradable mutex__ 를 제공한다. 정확하게 표현하면, upgradable mutex는 mutex의 가능성을 판단하여 여러 개의 쓰레드가 동시에 락을 공유할 수 있도록 만든 기법이다. 이러한 기법을 __multi reader and single write__ 기술이라고 말한다. 읽기는 데이터 수정이 발생되지 않기 때문에 여러 개의 읽기 연산이 수행되어도 공유하는 데이터에 영향을 주지 않는다. 반면, 쓰기 연산은 공유하고 있는 데이터 수정이 발생하기 때문에 다른 쓰레드에서 읽기 또는 쓰기 연산과 충돌되어 데이터 일관성 문제가 발생한다. __즉 쓰기 연산은 배타적으로 락이 설정되어야 한다.__
Upgradable mutex는 이와 같이 읽기 연산은 쓰기 연산이 없는 동안, 아무런 제재 없이 읽기가 원활하게 이루어져야 하고, 쓰기 연산은 다른 쓰레드들의 읽기 또는 쓰기 연산이 자신의 쓰기 연산이 완료될 때까지 사용할 수 없도록 만들어 준다.

쓰기 연산을 수행하기 위한 절차를 생각해 보자. 쓰기 연산은 공유 데이터에 접근하기 위해 현재 공유 데이터를 사용하고 있는 쓰레드들이 있는지 판단한다. 만약 공유 데이터를 사용하고 있는 쓰레드가 있다면, 해당 쓰레드가 사용을 완료할 때까지 기다려야 한다. 자신을 제외한 모든 쓰레드가 공유 데이터의 사용을 완료하였다면, 대기 중이던 쓰기 연산은 배타적 락을 설정하여 다른 쓰레드들이 자신의 작업이 완료될 때까지 공유 데이터에 접근할 수 없도록 만든다.

반대로, 읽기 연산은 현재 공유 데이터를 액세스하기 위해 설정된 락들을 우선 조사하여야 한다. 공유 데이터에 설정된 락이 모두 읽기 락이라면 읽기 연산은 바로 공유 데이터에 접근할 수 있다. 하지만, 다른 쓰레드가 설정한 락이 쓰기 연산을 위한 배타적 락이라면, 읽기 연산은 배타적 락이 해제될 때까지 기다려야 한다.

Upgrade mutex는 쓰기 연산과 읽기 연산을 위해 3가지 유형의 락을 제공한다.

### 배타적 락 exclusive lock
배타적 락은 일반 mutex와 비숫하다. 임의의 한 쓰레드가 배타적 락을 획득했다면, 다른 쓰레드들은 어떠한 락도 이미 획득된 배타적 락이 해제되지 않는다면 획득할 수 없다. 임의의 쓰레드가 공유 또는 업그레이드 락을 가지고 있다면, 배타적 락을 획득하려는 쓰레드는 블록된다. __배타적 락은 데이터를 수정하기 위한 쓰레드에서 사용한다.__

### 공유 락 Sharable lock
임의의 쓰레드가 공유 락을 획득하였다면, 다른 쓰레드들은 공유 또는 업그레이드 락을 획득할 수 있다. 만약 다른 쓰레드가 배타적 락을 가지고 있다면, 공유 락을 획득하기 위해 시도하는 쓰레드는 블록된다. __공유 락은 데이터를 읽기 위한 쓰레드에서 사용된다.__

### 업그레이드 락 Upgradable lock
임의의 쓰레드가 업그레이드 락을 획득하였다면, 다른 쓰레드들은 공유 락을 획득할 수 있다. 만약 다른 쓰레드가 배타적 또는 업그레이드 락을 가지고 있다면, 업그레이드 락을 획득하려는 쓰레드는 블록된다. 업그레이드 락은 업그레이드 락을 설정한 쓰레드는 공유 락을 획득한 다른 쓰레드들이 락을 모두 해제하면 자동으로 배타적 락을 획득할 수 있도록 보장해 준다. 업그레이드 락은 일반적으로 쓰레드들이 읽기를 수행하지만, 데이터를 수행하기를 원하는 쓰레드에서 사용할 수 있다. 즉, __쓰기를 수행하는 쓰레드는 업그레이드 락을 사용하고, 읽기를 수행하는 쓰레드는 공유 락을 사용한다.__ 한가지 주의할 점은 업그레이드 락은 배타적 락과 같이 오로지 한 개의 쓰레드에서만 획득할 수 있다.

## 몽고디비의 락 시스템
앞 절에서도 논하였지만, MongoDB는 boost의 shared_mutex를 기반으로 락 시스템을 만들었다. 따라서, __읽기는 동시에 여러 쓰레드에서 동시에 수행되지만, 쓰기는 한 개의 쓰레드에서만 수행된다.__ 이러한 락 시스템은 다음과 같은 특징을 가진다.

+ 읽기 락은 쓰기 락이 설정되지 획득되지 않은 이상 병렬로 처리되지만, 쓰기 락이 획득되면 대기 상태로 빠진다. 쓰기 락이 해제되면 읽기 연산이 수행된다.
+ 쓰기 락은 읽기 락이 모두 해제된 상태에서 획득되며, 배타적 락을 형성한다.

따라서, __MongoDB는 쓰기와 읽기가 동시에 수행되지 않는다.__ 만약 시스템이 읽기를 많이 시도할 경우에는 쓰기가 블록 되고, 쓰기를 많이 수행할 경우는 읽기가 블록 된다.

![그림 3-1](./images/pic3-1.png)
[그림 3-1]은 MongoDB를 복제 시스템으로 구성하고 클라이언트가 읽기 연산을 수행하는 상태를 보여주고 있는 것으로 Slave_OK를 이용하여 master와 slave 모두 읽기가 가능한 상태를 나타낸다. 따라서 MongoDB는 master와 slave 모두 읽기 연산을 요청한다.  

(a)는 master와 slave 모두 락이 설정된 것이 없기 때문에 바로 읽기 연산 결과를 리턴 한다. __MongoDB 클라이언트는 master와 slave 둘 중에 하나가 먼저 리턴 된 값을 받아 그 결과를 응용에 전달한다.__  

(b)의 경우는 master와 slave 모두 읽기 락이 획득되어 있지만, __읽기 락이 공유 락이기 때문에 (a)와 같이 바로 연산의 결과 값을 리턴 한다.__

![그림 3-2](./images/pic3-2.png)
[그림 3-2]는 [그림 3-1]의 확장된 형태로 master와 slave에 쓰기 락이 구성될 때를 보여준다.  

(a)는 master에 쓰기 락이 설정된 것으로 클라이언트가 보낸 읽기 연산이 블록 된다. 따라서, 클라이언트는 slave로부터 전달 받은 읽기 연산 결과를 먼저 받게 되고, 응용에 해당 결과를 통보한다. __속도는 slave의 읽기 연산 속도와 동일하며, 쓰기 락에 따른 지연은 발생되지 않는다.__   

(b)는 조금 다른 상황이다. Master와 slave 모두 쓰기 락이 획득한 상태이다. __이러한 경우는 master와 slave로 보낸 읽기 연산 모두가 블록 되기 때문에, 읽기 연산이 지연된다.__ Slave의 쓰기 락은 Oplog의 동기화 또는 slave의 인덱스 재생성(foreground indexing) 때 발생된다. 만약 동기화 데이터가 많을 경우는 master의 락이 먼저 해제되어 master로 부터 읽기 연산의 결과를 받아 들일 것이다. 일반적으로 쓰기 연산은 벌크 삽입(bulk insert)인 경우를 제외하고 장시간을 요하지는 않는다. 따라서, 일반 slave의 Oplog 동기화를 위한 쓰기 락 보다 master의 쓰기 락이 먼저 해제 되는 것이 일반적이다.

## 몽고디비의 Global Lock
MongoDB의 global lock은 용어에 풍겨 나오는 이미지에 의해 여러 개의 노드로 구성된 MongoDB 시스템에서 유일하게 사용되는 전체 락 이라고 생각할 수 있지만, 이는 전체 락 이기 보다는 __MongoDB 시스템의 락 정보를 저장하고 있는 저장소이다.__ 저장하는 정보로는 현재 락 상태, 락 지속 여부, 현재 대기 중인 읽기 또는 쓰기 연산 수, 그리고 클라이언트 개수가 있다. [표 3]는 MongoDB의 serverStatus를 통해 취득할 수 있는 global lock의 항목을 보여준다.

[표 3] Global lock 항목

|항목|	내용|
|---|-------|
|globalLock.totalTime|	Global lock이 생성된 시간으로 시스템이 기동되고 난부터 수행중인 시간을 의미하다.(단위 : microseconds)|
|globalLock.lockTime|	Lock 지속시간을 나타내는 것으로, 프로세스가 기동하고 난 다음부터 현 시점까지 lock이 걸려있던 시간의 합을 의미한다.(단위 : microseconds)|
|globalLock.ratio|	lockTime / totalTime 의 값|
|globalLock.currentQueue.total|	currentQueue.readers + currentQueue.writers 의 값|
|globalLock.currentQueue.readers|	연산 중에서 lock이 걸린 읽기 연산의 개수|
|globalLock.currentQueue.writers|	연산 중에서 lock이 결린 쓰기 연산의 개수|
|globalLock.activeClients.total|	activeClients.readers + activeClients.writers 의 값|
|globalLock.activeClients.readers|	읽기 연산을 수행하고 있는 클라이언트의 개수|
|globalLock.activeClients.writers|	쓰기 연산을 수행하고 있는 클라이언트의 개수|

[표 3]의 값에서 주목하여야 할 값은 `lockTime`이다. `lockTime`은 MongoDB가 쓰기 락이 수행된 시간을 누적시킨 값으로, 값의 증가가 빨라지면 lockTime이 길어진다는 의미이다. 이 값이 길어지게 되는 원인은 여러 가지 발생할 수 있는데, 대부분 시스템 부하가 발생되어 lockTime이 길어진 것으로 보아야 한다. 한가지 분명한 사실은 쓰기 연산이 발생할 때마다 lockTime은 증가하며, __전체 시간 대비 lockTime이 증가하면 시스템 부하가 발생하였다고 예측할 수 있다.__

[표 3]의 `currentQueue`는 __MongoDB가 쓰기 락이 수행될 경우, 쓰기 락에 의해 대기 중인 연산을 의미한다.__ 명칭 큐로 설정되어 있어서 MongoDB가 큐에 연산을 넣고 해당 연산을 빼내어 처리한다고 판단할 수 있으나, 소스 코드상으로 볼 때, 연산을 요청한 클라이언트 리스트에서 해당 정보를 취득한다. 즉, 클라이언트가 요청할 때, 한 개의 쓰레드가 생성되는데, 이러한 쓰레드 리스트 중에서 현재 락이 걸린 연산들의 개수를 리턴 한다.

[표 3]의 `activeClients`는 __연산을 요청한 클라이언트의 개수를 저장__ 하는 것으로, 읽기 연산과 쓰기 연산을 요청한 클라이언트의 개수가 된다. 클라이언트의 한 개의 요청은 서버의 한 개의 쓰레드를 만들어 클라이언트의 요청이 할당된다. 서버에 쓰기 락이 활성화되지 않았다면, 서버가 생성한 쓰레드는 락이 걸리지 않는다. `cuurentQueue`와 `activeClients`는 모두 동일한 서버의 클라이언트 리스트에서 정보를 취득하지만, 임의의 클라이언트가 락이 걸려 있는지에 따라 데이터 값의 차이가 나타난다. 따라서, `currentQueue`의 값이 activeClients의 값 보다 클 수 없다.

MongoDB의 구성 요소인 mongos는 mongod와 달리 클라이언트의 역할을 담당하므로, `lockTime`, `currentQueue`, `activeClients`의 값이 나타나지 않는다. Global Lock은 서버 관점에서 클라이언트가 요청한 연산에 대한 락 상태를 보여주는 것이다. 따라서, [표 3]의 값은 mongod에서만 유효하다. 또한, [표 3]는 각각의 mongod에 따라 다른 값이 나타난다. MongoDB가 말하는 Global Lock은 한 개의 mongod 프로세스에서 발생하는 값을 의미한다. 만약 여러 개의 노드로 구성된 시스템에서의 Global Lock의 총합을 알고자 한다면, 각 mongod 프로세스에서 [표 3]의 globalLock 값을 취득한 뒤, 합쳐야 한다.

MongoDB는 메모리 데이터베이스의 성격을 가지고 있다. 따라서, 메모리에 데이터를 쓰는 시간 만큼의 쓰기 락이 주어진다. 이는 MongoDB의 일관성 정책이 NORMAL인 경우에 해당된다. 만약 일관성이 NORMAL이 아닌 SAFE라면 쓰기 락이 시간도 같이 올라간다.