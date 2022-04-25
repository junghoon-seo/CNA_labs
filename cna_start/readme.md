


# [구현] 마이크로서비스의 실행


Instruction
> 누락된 유틸리티 설치


```
apt-get update
apt-get install net-tools
```

> 제대로 설치된 경우 Labs > 포트확인 클릭하여 포트넘버 확인 가능해야 합니다.

### 생성된 마이크로 서비스들의 기동
##### 터미널에서 mvn 으로 마이크로서비스 실행
```
cd order
mvn spring-boot:run
```
##### IDE에서 실행
* order 서비스의 Application.java 파일로 이동한다.
* 14행과 15행 사이의 'Run’을 클릭 후, 5초 정도 지나면 서비스가 터미널 창에서 실행된다.
* 새로운 터머널 창에서 netstat -lntp 명령어로 실행중인 서비스 포트를 확인한다.

##### 서비스 테스트
* 기동된 order 서비스를 호출하여 주문 1건을 요청한다.
```
http localhost:8081/orders productId=1 productName="TV" qty=3
```
* 주문된 상품을 조회한다.
```
http localhost:8081/orders
```
* 주문된 상품을 수정한다.
```
http PATCH localhost:8081/orders/1 qty=10
```
##### IDE에서 디버깅
1. OrderApplication.java 를 찾는다, main 함수를 찾는다.
2. main 함수의 첫번째라인 (16) 의 왼쪽에 동그란 breakpoint 를 찾아 활성화한다
3. main 함수 위에 조그만 "Debug"라는 링크를 클릭한다. (10초 정도 소요. 기다리셔야 합니다)
4. 잠시후 디버거가 활성화되고, 브레이크 포인트에 실행이 멈춘다.
5. Continue 라는 화살표 버튼을 클릭하여 디버거를 진행시킨다.
6. 다음으로, Order.java 의 첫번째 실행지점에 디버그 포인트를 설정한다:
```
@PostPersist
    public void onPostPersist(){
        OrderPlaced orderPlaced = new OrderPlaced();  // 이부분
        BeanUtils.copyProperties(this, orderPlaced);
        orderPlaced.publishAfterCommit();
    }
```    
1. 그런다음, 앞서 주문을 넣어본다
2. 위의 Order.java 에 디버거가 멈춤을 확인한후, variables 에서 local > this 객체의 내용을 확인한다.

### 실행중 프로세스 확인 및 삭제
netstat -lntp | grep :808 
kill -9 <process id>

##### 상세설명

https://www.youtube.com/watch?v=gtBQ9WFAbUQ
https://www.youtube.com/watch?v=J6yqEJrQUyk

