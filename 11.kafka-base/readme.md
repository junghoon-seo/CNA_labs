Instruction
Kafka Pub/Sub
토픽생성
/usr/local/kafka/bin/kafka-topics.sh --bootstrap-server http://localhost:9092 --topic topic_example --create --partitions 1 --replication-factor 1
토픽 리스트 보기
/usr/local/kafka/bin/kafka-topics.sh --bootstrap-server http://localhost:9092 --list    
새로운 터미널 창에서 kafka producer 연결 후 메세지 publish
/usr/local/kafka/bin/kafka-console-producer.sh --broker-list http://localhost:9092 --topic topic_example
새로운 터미널 창에서 kafka consumer 연결 후 메세지 subscribe
/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server http://localhost:9092 --topic topic_example --from-beginning
hello world 라는 메세지 publish 해보기
Kafka Consumer Group & Offsets
consumer group 목록
/usr/local/kafka/bin/kafka-consumer-groups.sh --bootstrap-server http://localhost:9092 --list
consumer group 의 offset 확인
/usr/local/kafka/bin/kafka-consumer-groups.sh --bootstrap-server http://localhost:9092 --group <group_id> --describe
consumer group 의 offset 재설정
/usr/local/kafka/bin/kafka-consumer-groups.sh --bootstrap-server http://localhost:9092 --group <group_id> --topic topic_example --reset-offsets --to-earliest --execute
There are many other resetting options, run kafka-consumer-groups for details:

 --shift-by <positive_or_negative_integer>
 --to-current
 --to-latest
 --to-offset <offset_integer>
 --to-datetime <datetime_string>
 --by-duration <duration_string>
