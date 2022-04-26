> Instruction
## 자바 애플리케이션의 패키징
* 터미널을 열어서 order 와 delivery, gateway 폴더로 각각 이동하여 아래 명령어를 실행한다.
```
cd shopmall
cd order
mvn package -B -Dmaven.test.skip=true
```

* target 폴더에 jar 파일이 생성이 되었는지 확인한다.
```
java -jar target/order-0.0.1-SNAPSHOT.jar
```
> 명령으로 실행이 가능한지 확인한다.

* ctrl+c 를 눌러서 jar 실행에서 빠져 나온다.

## 도커라이징
* order 와 delivery, gateway 의 최상위 root 에 Dockerfile 이 있는지 확인 한다.
* Dockerfile 파일이 있는 경로에서 아래 명령을 실행 한다.
```
docker login
docker build -t [dockerhub ID]/order:[오늘날짜] .     
docker images
docker push [dockerhub ID]/order:[오늘날짜]  
```

## 클러스터에 배포
### yaml 파일로 배포
* order/kubernetes 폴더내의 deployment.yaml을 오픈한다.
* image: 부분을 push 한 이미지 명으로 수정한다: [dockerhub ID]/order:[오늘날짜]
* 저장후, 다음명령:
```
kubectl apply -f kubernetes/deployment.yml

kubectl apply -f kubernetes/service.yml
```

### 명령으로 배포 (비추)
* order 서비스 배포
```
kubectl create deploy order --image=[dockerhub ID]/order:latest
kubectl expose deploy order --port=8080
```
* delivery 서비스 배포
```
kubectl create deploy delivery --image=[dockerhub ID]/delivery:latest
kubectl expose deploy delivery --port=8080
```
* gateway 서비스 배포
```
kubectl create deploy gateway --image=[dockerhub ID]/gateway:latest
kubectl expose deploy gateway --type=LoadBalancer --port=8080
```
## 서비스 확인
* 게이트웨이 주소 확인
```
kubectl get svc
```
* Pod 생성 확인
```
kubectl get po
```
* 주문 확인
```
http [gateway IP]:8080/orders
http [gateway IP]:8080/orders productId=1 productName="TV" qty=3
```
## 잘 안될때
* 쿠버네티스 객체 들이 이미 존재하는 경우, 다음을 통하여 객체들을 제거:
```
kubectl delete deploy --all
kubectl delete svc --all
```
### 더 많은 테스트
```
kubectl delete po --all
# 한후, 서비스 접속 -> 좀있다가 회복
kubectl get po   # po가 다시 생성되었음을 확인

kubectl scale deploy order --replicas=3
kubectl get po 
# order를 위한 pod가 3개가 생성됨을 확인
```

* 상세설명
