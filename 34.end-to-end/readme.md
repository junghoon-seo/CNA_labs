Instruction
개발기에서 전체 시스템 무작정 따라하기
프론트엔드
새 터미널을 열고 아래 입력:

git clone https://github.com/event-storming/ui.git
cd ui
npm install
npm run serve
Labs 메뉴 > 포트열기 > 8080
12 번가 메인 화면을 확인
백엔드 마이크로 서비스들이 필요하기 아무런 동작을 하지 않음
도커빌드 후 쿠버네티스에 반영하기

npm run build
docker build -t <계정명>/ui:stable .
docker push <이미지명>
상품 서비스
새 터미널을 열고 아래 입력:
cd ~ 
git clone https://github.com/event-storming/products.git
cd products
mvn spring-boot:run
서비스가 모두 올라올때까지 대기
새 터미널을 열고 다음명령 입력:
http localhost:8085/products
상품 정보들을 확인:
{
  "_embedded" : {
    "products" : [ {
      "@id" : 1,
      "name" : "TV",
      "price" : 10000,
      "stock" : 10,
      "imageUrl" : "/goods/img/TV.jpg",
      "_links" : {
        "self" : {
          "href" : "http://labs-2091653521:8085/products/1"
        },
        "product" : {
          "href" : "http://labs-2091653521:8085/products/1"
        },
        "productOptions" : {
          "href" : "http://labs-2091653521:8085/products/1/productOptions"
        }
      }
    }, {
      "@id" : 2,
      "name" : "MASK",
      "price" : 20000,
      "stock" : 20,
      "imageUrl" : "/goods/img/MASK.jpg",
      "_links" : {
        "self" : {
          "href" : "http://labs-2091653521:8085/products/2"
        },
        "product" : {
          "href" : "http://labs-2091653521:8085/products/2"
        },
        "productOptions" : {
          "href" : "http://labs-2091653521:8085/products/2/productOptions"
        }
      }
    },
    ...
도커빌드:

mvn package -B
docker build -t <계정명>/product:stable .
쿠버네티스 yaml 만들기

apiVersion: "apps/v1"
kind: "Deployment"
metadata: 
  name: "products"
  labels: 
    app: "products"
  namespace: "shopping"
spec: 
  selector: 
    matchLabels: 
      app: "products"
  replicas: 1
  template: 
    metadata: 
      labels: 
        app: "products"
    spec: 
      containers: 
        - 
          name: "products"
          image: "jinyoung/products:stable"
          ports: 
            - 
              containerPort: 8080

API 게이트웨이
새 터미널을 열고
cd ~
git clone https://github.com/event-storming/gateway.git
cd gateway/
mvn spring-boot:run
메뉴 Labs > 포트 열기 > 8088
게이트웨이 주소 확인
e.g. http://8088-labs-2091653521.kuberman.io
인증서버가 아직 미구동이기 떄문에 HTTP ERROR 401 에러가 떠있음
브라우저가 열린 주소 뒤에 /products 입력 (/products path 는 인증서버를 안타도록 구현됨)
e.g. http://8088-labs-2091653521.kuberman.io/products
프론트엔드와 API 게이트웨이 연결
프론트엔드 서버 내리기
프론트엔드 서버를 실행했던 터미널에서 Ctrl + C
환경변수에 API 게이트웨이 정보를 설정
npm run build
docker build -t frontend .
docker run -d -p8080:8080 -e VUE_APP_API_HOST='<얻어낸 API 게이트웨이 주소>' frontend 
주의사항 : VUE_APP_API_HOST 주소 입력시 “/” (역슬레쉬) 가 들어가지 않도록 주의!!!
역슬레쉬 가 들어갔을때 CORS 에러가 뜰수 있으니, 혹시 CORS 오류시 export 를 새로 하고 dokcer build 를 새로 해야함.
만약 새로 할 경우에는 docker ps 명령으로 현재 사용중인 도커 이미지 확인
docker stop [CONTAINER ID]
docker run -d -p8080:8080 -e VUE_APP_API_HOST=’<얻어낸 API 게이트웨이 주소>’ frontend

image

주문 서비스
새 터미널을 열고
cd ~
git clone https://github.com/event-storming/orders.git
cd orders
mvn spring-boot:run
배송 서비스
새 터미널을 열고
cd ~
git clone https://github.com/event-storming/delivery.git
cd delivery
mvn spring-boot:run
프론트엔드 브라우저에서 주문 하여 보기
프론트엔드에 접속하여 웹 브라우저 리프래시
메뉴 Labs > 포트 열기 > 8080
로그인 화면에서, 로그인 버튼 클릭
로그인이 안됨. --> 인증서비스 필요!

인증 서비스
새 터미널을 열고
cd ~
git clone https://github.com/event-storming/oauth.git
cd oauth
mvn spring-boot:run
다시 로그인 하여 주문시도
프론트엔드에서 로그인 버튼 클릭후
상품조회 화면에서 TV 2대를 주문 해본다.
시스템간 연동의 확인
주문한 상품의 재고 수량이 변경됨을 확인:
http localhost:8085/products

{
  "_embedded" : {
    "products" : [ {
      "@id" : 1,
      "name" : "TV",
      "price" : 10000,
      "stock" : 8,
      "imageUrl" : "/goods/img/TV.jpg",
      "_links" : {
        ...
      }
    }, 
    ...
TV 의 stock 이 10개에서 8개로 줄어든 것을 확인할 수 있다.

배송상태를 조회

http localhost:8082/deliveries
주문건들에 대한 배송 데이터가 출력됨을 확인
배송 서비스 장애 상황과 장애 격리 확인
배송서비스 터미널에서 Ctrl + C 하여 배송서비스를 멈춤
주문화면에서 마스크를 5개 구매
주문상태 페이지에서 배송내역 확인 - 배송상태 표시 안됨
배송이 안되더라도 주문은 문제가 없음을 확인하는 것이 포인트!

배송 서비스 복구시 최종 일관성 (Eventual Consistency)에 의해 데이터가 일치됨을 확인
배송 서비스를 다시 기동
배송상태를 조회
http localhost:8082/deliveries
주문건들에 대한 누적된 배송 데이터가 출력됨을 확인
운영단계 무작정 따라하기
개발기 프로세스 모두 종료하기
모든 터미널 정삭적으로 닫기
모든 터미널에 접속한 후, Ctrl+C 입력
정상종료 후 다시 명령 프롬프트 상태가 되면 터미널 창을 닫기
상품서비스와 API Gateway 를 쿠버네티스에 배포하기
새로운 터미널 열기, 다음을 입력:
kubectl run products --image=jinyoung/12mall-product:latest
kubectl expose deploy products --port=8080

kubectl run gateway --image=jinyoung/12mall-gateway
kubectl expose deploy gateway --port=8080 --type=LoadBalancer
게이트웨이를 프론트엔드와 연결하기위해 게이트웨이의 주소를 복사:
kubectl get svc -w
얻어낸 EXTERNAL IP 를 복사, 브라우저에서 접속해보았을때 상품정보 json 이 출력되는지 확인:
e.g. http://35.243.124.94:8080/products
