---
layout: post
title:  "Redis Partitioning"
date:   2018-10-13 17:44:00 +0900
categories: docs redis
---

comes from [here][redis-partitioning]

## Partitioning 이론

### Partitioning in Redis serves two main goals: 레디스의 파티셔닝의 두 가지 주목적
> * It allows for much larger databases, using the sum of the memory of many computers. Without partitioning you are limited to the amount of memory a single computer can support.
> * It allows scaling the computational power to multiple cores and multiple computers, and the network bandwidth to multiple computers and network adapters.

1. 여러 대의 컴퓨터의 메모리를 이용해서 더 큰 저장소로 이용할 수 있고
2. 다수의 코어, 컴퓨터를 이용하여 확장해 갈 수 있습니다.

### Partitioning basics 기본 파티셔닝

여러가지 파티셔닝 방법이 있습니다. 예를 들면 레디스 인스턴스 *R0, R1, R2, R3* 가 있고 사용자들의 키 _user:1, user:2_ 가 있을 때 어떤 인스턴스가 어떤 키를 저장하고 있는지를 판별하는 방법이 여러가지가 있습니다.

가장 단순한 방법 중에 하나는 **range partitioning** 입니다. 어떤 키가 어떤 인스턴스에 저장되어야 하는지 mapping range 정하는 것입니다. 예를 들어, user 1~10000 는 *R0* 에, user 10001~20000 는 *R1* 에 저장합니다.

잘 동작하고 실제 이용되는 방법이기도 하지만 mapping 정보를 별도로 존재해야 한다는 점이 단점입니다. mapping 정보를 따로 관리해야 하고 저장하는 객체마다 해당 정보가 필요합니다. 그래서 다른 파티셔닝 방법에 비해 효율성이 떨어져 권장할 만한 방식은 아닙니다.

다른 대체 가능한 방법은 **hash partitioning** 입니다. 키의 형태는 몰라도 되며 아무 형식의 키라도 상관없습니다.
* 키를 해쉬 함수(crc32 같은) 을 이용하여 숫자로 변환합니다. 예를 들어 _foobar_ 같은 키가 _93024922_ 같은 숫자로 변환됩니다.
* modulo 연산을 이용해서 어떤 인스턴스에 저장할지 정합니다. 예를 들어 *R0~R3* 4개의 인스턴스가 있으므로 `93024922 modulo 4` 를 하여 R2 에 저장합니다.

hash partitioning 조금 더 진보된 방식으로 **consistent hashing** 이 있습니다. 일부 레디스 클라이언트 및 프록시에서 사용하고 있는 방식입니다.
> ##### 이곳에는 consistent hashing 에 대한 별도 설명이 없습니다.

### Different implementations of partitioning 파티셔닝의 여러가지 구현방법

* **Client side partitioning** 은 클라이언트가 바로 인스턴스를 선택하는 방식입니다. 많은 레디스 클라이언트에서 이 방식을 사용합니다.
* **Proxy assisted partitioning** 은 클라이언트가 레디스 프로토콜을 알고 있는 프록시에게 요청을 보냅니다. 프록시에서는 설정된 파티셔닝 설정에 따라 쓰기/읽기 되어야 할 인스턴스에게 전달하고 클라이언트에게 결과를 전송합니다. [Twemproxy][twemproxy-link] 에 구현된 방식입니다.
* **Query routing** 이란 실행할 명령어를 랜덤하게 선택된 인스턴스로 보내고 해당 인스턴스가 올바른 인스턴스를 선택하는 방식입니다. *Redis Cluster* 에서 Query routing 을 조합한 방식을 구현하고 있습니다. 인스턴스에서 인스턴스로 보내지 않고 인스턴스에서 클라인언트로 전달하여 클라이언트에서 최종 인스턴스에게 전달하는 방식입니다.

### Disadvantages of partitioning 파티셔닝의 단점
> ##### 레디스에서 파티셔닝 사용 시 발생할 수 있는 단점들

* multiple keys 명령들은 대개 지원하지 않습니다. 예를 들면 두개의 set 에 대해 서로 다른 인스턴스에 저장되어 있는 경우 교집합을 구할 수 없습니다. (할 수는 있지만 직접적인 방법은 없습니다)
* multiple kyes 에 대한 Redis transaction 을 지원하지 않습니다.
* 파티셔닝은 키를 기반이므로 하나의 키가 대량의 정보를 가지고 있는 경우는 분할될 수 없습니다.
* 파티셔닝 이용 시에는 데이터 관리가 쉽지 않아 집니다. 다수의 RDB / AOF 파일을 관리해야 하므로 데이터 백업을 위해서는 여러 인스터스, 호스트로 나눠져 있는 파일을 하나로 합쳐야 하는 어려움이 있습니다.
* 인스턴스 추가/삭제가 복잡할 수 있습니다. Redis Cluster 는 노드의 추가/삭제로 인한 데이터 리밸런싱의 투명성을 대부분 보장하지만 클라이언트 측의 파티셔닝이나 프록시의 경우 해당 기능을 보장하지 않습니다. 하지만 _Pre-sharding_ 기법으로 해결될 수 있습니다.

### Data store or cache? 데이터 저장소로 사용될 때와 캐시로 사용될 때
데이터 저장소로 이용되거나 캐시로 이용되거나 파티셔닝 방식에는 차이가 없습니다. 하지만 데이터 저장소로 이용될 때에는 어떤 주어진 키에 대해서 항상 동일한 인스턴스로 연결되어야 한다는 제한이 있습니다. 캐시로 이용될 때는 특정 노드가 사용불가한 상태라도 큰 문제가 되지 않습니다. 그냥 다른 노드를 이용하면 되고 이런 키 - 인스턴스 매핑 정보의 변경은 시스템의 가용성(availability)을 높여주는 방식입니다.
> ##### availability : 요청온 명령에 응답하는 능력

*Consistent hashing* 방식에서는 필요한 노드하나가 사용 가능하지 않을 때 다른 노드로의 변환이 발생합니다. 새로운 노드가 추가되었을 때도 일부 새로운 키가 새로운 노드에 저장되게 됩니다.

* 캐시로 이용될 때에는 _consistent hashing_ 을 이용한 **확장이 쉽습니다.**
* 데이터 저장소로 이용될 때는 키-인스턴스 맵을 고정하므로 노드의 갯수도 고정되어야 하므로 해당 매핑 정보는 **변경될 수 없습니다.** 꼭 변경해야 한다면 노드가 추가/삭제 될 때 키에 대한 리밸런싱 작업이 필요합니다. 현재 Redis Cluster 만 가능한 기능이며, Redis Cluster 는 2015년 4월 1일부터 사용 가능한 상태입니다.

### Presharding 프리샤딩
위에서 이야기했다시피 레디스를 데이터 저장소로 이용하는 경우 노드의 추가/삭제가 까다로운 작업이며 고정된 키-인스턴스 맵 정보를 사용하는 게 제일 단순한 방법임을 알게 되었습니다.

하지만 데이터 저장소 이용할 때도 변동이 생깁니다. 오늘은 10개의 노드만 필요했지만 내일은 30개의 노드가 필요할지도 모르지요.

레디스는 초초경량이므로(여유분의 인스턴스가 오직 1MB만 사용합니다.), 가장 단순한 접근 방법은 처음부터 많은 량의 인스턴스로 시작하는 것입니다. 하나의 서버만 있다고 하더라도 파티셔닝을 이용하여 분산처리 하기로 마음먹을 수도 있습니다. 그리고 처음부터 확장성을 고려해 32/64 개의 넉넉한 인스턴스를 확보하는 것입니다.

이렇게 하는 경우 데이터 저장소의 확장이 필요한 경우 그저 기존에 만들어진 인스턴스를 다른 레디스 서버로 이동할 수 있습니다. 새로운 레디스 서버가 추가되면 기존의 절반의 인스턴스를 옮길 수 있습니다.

Redis replication 을 이용하는 경우 최소한의 혹은 서비스 중단 없이 데이터를 이동할 수 있습니다.

## Implementations of Redis partitioning 파티셔닝 구현
> ##### 그래서 어떤 거 쓰라는...?

### Redis Cluster
Redis Cluster 에서는 자동 샤딩과 높은 가용성을 얻을 수 있습니다. Redis Cluster 는 _query routing_ 과 _client side partitioning_ 방식을 혼합해서 사용하고 있습니다. [Cluster tutorial][redis-cluster-tutorial]

### Twemproxy
트위터에서 개발한 멤캐시와 레디스 프로토콜을 위한 프록시 입니다. 싱글 스레드로 운영되고 C로 작성되어 있으며 완전 빠릅니다. (Apache 2.0 license)

Twemproxy 는 여러대의 레디스 인스턴스에 대해 자동 샤딩을 지원하며 노드가 사용가능하지 않을 때 노드 제거 기능을 제공합니다. 이 기능은 키-인스턴스의 변경이 일어나므로 레디스를 캐시로 이용할 때만 이용해야 하는 기능입니다.

기본적으로 Twemproxy는 클라이언트와 레디스 인스턴스를 이어주는 중간 다리역할을 합니다. 그 중간에서 파티셔닝을 제어합니다. [Read more][twemproxy-more]

### Consistent hashing 을 제공하는 클라이언트
Twemproxy을 대체할 만한 다른 방법은 _consistent hashing_ 이나 비슷한 알고리즘을 구현한 다른 Redis Client 를 이용하는 것입니다. 예를 들면, `Redis-rb` and `Predis` 같은 것들이 있습니다. [이곳][redis-client]에서 _consistent hashing_ 을 구현한 클라이언트가 있는지 찾아 볼 수 있습니다.





[redis-partitioning]: https://redis.io/topics/partitioning
[twemproxy-link]: https://github.com/twitter/twemproxy
[twemproxy-more]: http://antirez.com/news/44
[redis-client]: https://redis.io/clients
[redis-cluster-tutorial]: https://redis.io/topics/cluster-tutorial
