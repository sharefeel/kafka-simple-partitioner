# kafka-simple-partitioner
Kafka Partitioner that does not hash with key

The DefaultPartitioner include in kafka-clients determines a partition to send record as
* Rule 1. If key is specified, statically choose a partition with murmurhash(byte array of key) % number of partitions
* Rule 2. Else choose a partition in round robin manner within available partitions.

This rule 1 can cause serious problem when key is given and there are unavailable partitions because of "% number of partitions". Not "% of number of AVAILABLE partitions". If caculation result is dead partition, that recored can not be deliver forever until dead partition becomes available. AND all queuing record of that producer also can not deliver until dead partitions resurrect. Of course, pending can be spread over all producer.

Yes. Purpose of DefaultParitioner's key is sharding. But if you want key as other purpose like:
* Both key and value are data but serialization class are different. For example, value is string and key is avro object.
* Key is header describes it's value. You can obtain information of value without full parsing of value.

To deliver without failures, rule 1 must excluded in choosing a partition logic regardless of key existence. SimpleParitioner choose a partition with only rule 2. Just send record to available partition.
