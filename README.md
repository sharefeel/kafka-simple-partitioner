# kafka-simple-partitioner

This partitioner tested on 0.8.2

SimplePartitioner는 메세지 전송할 파티션을 가용한 파티션 중 round-robin으로 선택하는 파티셔너이다.

카프카 프로듀서에 기본으로 사용되는 DefaultPartitioner는 레코드(key, value pair)를 전송할 파티션을 다음과 같이 결정한다.

* Rule 1. Key가 주어지는 경우, 다음 식을 수행해서 파티션을 선택: murmurhash(byte array of key) % number of partitions.
  파티션의 수가 변하지 않는 이상 key에 따라 전송되는 파티션은 static하게 선택됨
* Rule 2. 반대로 key가 주어지지 않는 경우, 가용한 파티션 중 round-robin manner로 파티션을 선택

Rule 1 로 인해 producer 전체가 먹통이 되는 심각한 상황이 만들어질 수 있다.
그 이유는 rule 1 으로 인해 리턴되는 파티션의 번호에 가용하지 않은 파티션 번호가 포함되어 있기 때문이다.
Static한 hashing으로 파티션을 선택하므로 문제있는 파티션이 다시 가용해지기 전에는 파티션으로의 메세지 전송이 불가능하다.
더해서 프로듀서는 메세지 전송큐는 하나이기 때문 하나의 전송이 실패하면 재전송을 시도하는 동안 모든 파티션으로의 전송이 딜레이된다.
(이 문제는 critical 하다고 생각하는데 0.8.2 기준이므로 이후 버전에는 수정되었을 수 있다.)

사실 replica를 늘림으로써 쉽게 문제를 약화시킬 수 있다. 위 문제는 어떤 이유에서는 가용하지 않은 파티션이 생긴 상황을 가정한 것이다.

눈치챘겠지만 DefaultPartitioner에서 key의 목적은 명백히 sharding이다.
하지만 sharding이 아니라 다음과 같은 목적으로 사용하고자 하는 경우에는 DefaultPartitioner의 동작은 문제의 소지가 있다.

* Key를 value를 설명하는 헤더로 사용하는 경우.
* Data 자체가 두 부분으로 분리되어 있고 serde 자체가 다른 경우. 예를 들어 data의 일부는 avro이고 나머지는 json string인 경우.

이런 경우 실패없는 메세지 전송을 보장하기 위해서는 파티션 선택방법에서 key를 배제해버리는 것이 최선이다.
SimplePartitioner는 rule 1 을 배제하고 rule 2만을 사용해 파티션번호를 리턴한다.

Kafka Partitioner that does not hash with key

The DefaultPartitioner included in kafka-clients determines a partition to send record(pair of key, value) as

* Rule 1. If key is specified, statically choose a partition with murmurhash(byte array of key) % number of partitions
* Rule 2. Else key is not give, choose a partition in round robin manner within available partitions.

This rule 1 can cause serious problem when key is given and there are unavailable partitions because of "% number of partitions". Not "% of number of AVAILABLE partitions". If record's hash caculation result points to dead partition, that record can not be delivered until dead partition becomes available. AND all queuing record of that producer also can not be delivered until dead partitions will resurrect.

Yes. Purpose of DefaultParitioner's key is clearly sharding. But if you want to use key as other purpose like:

* Data is consists of two different serializations and they must be sent together in one record.
* Key is a header and describes the record value. You can obtain information of value without full parsing of value.

To deliver without failures, rule 1 must be excluded in choosing a partition regardless of key existence. SimpleParitioner choose a partition based on only rule 2. Just send record to one of available partitions.
