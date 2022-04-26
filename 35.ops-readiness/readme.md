> Instruction
# 무정지 배포 실습 (readinessProbe 설정, 제로 다운타임)

* 이번 시간은 클러스터에 배포시 다운타임이 존재하는지 실습을 한다.
* 클러스터에 배포를 할때 readinessProbe 설정이 없으면 다운타임이 존재 하게 된다. 
* 이는 쿠버네티스에서 Ramped 배포 방식으로 무정지 배포를 시도 하지만, 
* 서비스가 기동하는 시간이 있기 때문에, 기동 시간동안 장애가 발생 할 수 있다.

* 배포시 다운타임의 존재 여부를 확인하기 위하여, siege 라는 부하 테스트 툴을 사용한다.
* 배포시작전에 부하테스트 툴을 실행하고, 배포 완료시 종료한 후, 결과값인 Availability 를 체크 하여 어느정도의 실패가 있었는지를 확인한다.

## 선행과정
* Kafka 가 설치되어있어야 한다:
> 설치가 필요한 경우,
> https://labs.msaez.io/#/courses/cna-full/running@sds-1012/ops-utility
> 를 참고하여 설치한다.

* 주문서비스를 배포한다:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order
  labels:
    app: order
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order
  template:
    metadata:
      labels:
        app: order
    spec:
      containers:
        - name: order
          image: jinyoung/order:stable
          ports:
            - containerPort: 8080
```
* 위의 파일을 deployment.yaml 파일로 만든후 저장한다.
* 저장한 파일을 배포한다:
```
kubectl apply -f deployment.yaml
```
* 서비스 객체를 위한 yaml도 만든다:
```
apiVersion: "v1"
kind: "Service"
metadata: 
  name: "order"
  labels: 
    app: "order"
spec: 
  ports: 
    - 
      port: 8080
      targetPort: 8080
  selector: 
    app: "order"
  type: "ClusterIP"
```
* 를 만든후
```
kubectl apply -f service.yaml
```
* 로 서비스를 한다.

### 부하 테스트 Pod 설치
* 아래 스크립트를 terminal 에 복사하여 siege 라는 Pod 를 생성한다.
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: siege
spec:
  containers:
  - name: siege
    image: apexacme/siege-nginx
EOF
```

* 생성된 siege Pod 안쪽에서 정상작동 확인
```
kubectl exec -it siege -- /bin/bash
siege -c1 -t2S -v http://order:8080/orders
exit
```
1. readinessProbe 가 없는 상태에서 배포 진행
1.1 새 버전을 배포할 준비를 한다:
* deployment.yaml 의 이미지 정보를 아래와 같이 변경한 후:

```
image: jinyoung/order:canary
```

1.2 새로운 터미널을 열어서 충분한 시간만큼 부하를 준다.
```
kubectl exec -it siege -- /bin/bash
siege -c1 -t60S -v http://order:8080/orders --delay=1S
```
1.3. 배포를 반영한다:
```
kubectl apply -f deployment.yaml
```

1.5 siege 로그를 보면서 배포시 정지시간이 발생한것을 확인한다.
```
Transactions:                     82 hits
Availability:                  70.09 %
Elapsed time:                  59.11 secs
```
2. readinessProbe 를 설정하고 배포 진행
2.1 아래와 같이 readiness 설정을 한다:
```
    spec:
      containers:
        - name: order
				  ...
          readinessProbe:    # 이부분!
            httpGet:
              path: '/orders'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
```
2.2. image 명도 변경한다:
```

	image: jinyoung/order:stable
```

2.2 siege 터미널을 열어서 충분한 시간만큼 부하를 준다.
```
kubectl exec -it siege -- /bin/bash
siege -c1 -t60S -v http://order:8080/orders --delay=1S
```
2.3 배포한다
```
kubectl apply -f deployment.yml
```

2.5 siege 로그를 보면서 배포시 무정지로 배포된 것을 확인한다.
```
Transactions:                    112 hits
Availability:                 100.00 %
Elapsed time:                  59.58 secs
```
> 주의점: siege 테스트를 걸어놓은 후 배포해야 정확한 테스트가 이루어집니다.

상세설명
