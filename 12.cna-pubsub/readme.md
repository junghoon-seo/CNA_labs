> Instruction
## 이벤트 Publish / Subscribe

* 마이크로 서비스간의 통신에서 이벤트 메세지를 Pub/Sub 하는 방법을 실습한다.
* Order 서비스에서 OrderPlaced 이벤트를 발행하였을때 Delivery 서비스에서 OrderPlaced 이벤트를 수신하여 작업 후 DeliveryStarted 를 발행한다.

### order 서비스의 이벤트 Publish
* order 마이크로 서비스를 8081 포트로 실행한다.
* order 서비스의 Application.java 파일로 이동한다.
* 14행과 15행 사이의 'Run’을 클릭 후, 5초 정도 지나면 서비스가 터미널 창에서 실행된다.
* 새로운 터머널 창에서 netstat -lntp 명령어로 실행중인 서비스 포트를 확인한다.
* 기동된 order 서비스를 호출하여 주문 1건을 요청한다.
```
http localhost:8081/orders productId=1 productName="TV" qty=3
```
* kafka Consumer에서 이벤트 확인
```
/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic shopmall --from-beginning
```

### delivery 서비스의 이벤트 Subscribe
* delivery PolicyHandler.java Code 확인한다.

* 주석을 해제 하고, delivery 서비스의 Application.java 파일로 이동한다.

* 14행과 15행 사이의 'Run’을 클릭 후, 5초 정도 지나면 서비스가 터미널 창에서 실행된다.

* OrderPlaced 이벤트에 반응하여 DeliveryStarted 이벤트가 연속적으로 발행되는 것을 확인한다.

* kafka Consumer에서 이벤트 메세지 확인
   > {“eventType”:“OrderPlaced”,“id”:1,“productId”:1,“qty”:3,“productName”:“TV”} <p>
   > {“eventType”:“DeliveryStarted”,“id”:1,“orderId”:1,“productId”:1,“productName”:“TV”}

### Service Clear
* 다음 Lab을 위해 기동된 모든 서비스 종료
```
  fuser -k 8081/tcp
  fuser -k 8082/tcp
```
