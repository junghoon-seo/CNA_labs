Instruction
컨테이너 오케스트레이션 무작정 따라해 보기
주문서비스 생성하기
도커 허브에 저장된 주문 이미지으로 서비스 배포 및 확인하기
kubectl create deploy order --image=jinyoung/monolith-order:v202105042
kubectl get all
Docker Hub에 Push한 나만의 이미지로 쿠버네티스에 배포해 보기
kubectl create deploy welcome --image=<내 이미지 full 주소>
kubectl get all : 생성된 객체(Pod, Deployment, ReplicaSet) 확인
kubectl get deploy -o wide : 배포에 사용된 이미지 확인
kubectl get pod -o wide : 파드가 생성된 워크노드 확인
트러블 슈팅
kubectl describe po [Pod 명]
kubectl logs -f [Pod 명]
kubectl exec -it [Pod 명] -- /bin/sh 
> ls
app.jar  dev      home     media    opt      root     sbin     sys      usr
bin      etc      lib      mnt      proc     run      srv      tmp      var

> ps
PID   USER     TIME  COMMAND
    1 root      0:38 java -Xmx400M -Djava.security.egd=file:/dev/./urandom -jar /app.jar --
   48 root      0:00 /bin/sh
   58 root      0:00 ps

오류가 있는 경우, 이를 수정한 버전을 올린다:

kubectl set image deploy order monolith-order=jinyoung/monolith-order:v20210504
주문서비스 삭제해 보기
# New Terminal (관측용)
watch kubectl get pod
# New Terminal (pod 삭제용)
kubectl delete pod -l app=order 
Pod를 삭제해도 새로운 Pod로 서비스가 재생성됨을 확인
클라우드 외부에서도 접근 가능하도록 노출하기
kubectl expose deploy order --type=LoadBalancer --port=8080
kubectl get service -w
External IP를 얻어오는데 오래걸리거나, ALB 등이 연결되는데, 시간이 걸리는 경우 다음의 port-forwarding 명령으로 localhost 에 접속할 수 있다:

# 새 터미널
kubectl port-forward deploy/order 8080:8080

# 다른 터미널
http localhost:8080
Service 정보의 External IP가 Pending 상태에서 IP정보로 변경시까지 대기하기
엔드포인트를 통해 서비스 확인 - http://(IP정보):8080/orders
Ctrl + C를 눌러 모니터링 모드 종료하기
접속테스트:

# http a78bb72215adc4a7c9db56a0c9acc457-1497647582.ap-northeast-2.elb.amazonaws.com:8080
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Wed, 26 May 2021 06:26:06 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "deliveries": {
            "href": "http://a78bb72215adc4a7c9db56a0c9acc457-1497647582.ap-northeast-2.elb.amazonaws.com:8080/deliveries{?page,size,sort}",
            "templated": true
        },
        "orders": {
            "href": "http://a78bb72215adc4a7c9db56a0c9acc457-1497647582.ap-northeast-2.elb.amazonaws.com:8080/orders{?page,size,sort}",
            "templated": true
        },
        "productOptions": {
            "href": "http://a78bb72215adc4a7c9db56a0c9acc457-1497647582.ap-northeast-2.elb.amazonaws.com:8080/productOptions"
        },
        "products": {
            "href": "http://a78bb72215adc4a7c9db56a0c9acc457-1497647582.ap-northeast-2.elb.amazonaws.com:8080/products{?page,size,sort}",
            "templated": true
        },
        "profile": {
            "href": "http://a78bb72215adc4a7c9db56a0c9acc457-1497647582.ap-northeast-2.elb.amazonaws.com:8080/profile"
        }
    }
}
주문서비스 롤백(RollBack) 하기
kubectl rollout undo deploy order
kubectl get deploy -o wide
주문서비스에 적용된 Image가 apexacme/order로 롤백되었음을 확인
주문서비스 인스턴스 확장(Scale-Out) 하기 (수동)
kubectl scale deploy order --replicas=3
kubectl get pod
주문서비스의 인스턴스(Pod)가 3개로 확장됨을 확인
YAML 기반 서비스 배포하기
Cloud IDE 메뉴 > File > Folder > YAML 입력
생성한 폴더 하위에 아래 파일 생성
Cloud IDE 메뉴 > File > New File > order.yaml 입력
아래 내용 복사하여 붙여넣기
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-by-yaml
  labels:
    app: order
spec:
  replicas: 2
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
          image: jinyoung/monolith-order:v20210504
          ports:
            - containerPort: 8080        
입력 후, 저장
- kubectl apply -f order.yaml 
- kubectl get all 
혹은, 기존의 쿠버네티스 내에 저장된 객체에서 yaml을 획득할수도 있다:

kubectl get deploy order -o yaml > order.yaml
하지만 얻어낸 객체의 status 파트와 uid 등 불필요한 속성들을 일일이 지워주고 계속사용해야 한다.

오류발생시
Already Exist:
root@labs-373904008:/home/project/ops-kubernetes# kubectl expose deploy order --type=LoadBalancer --port=8080
Error from server (AlreadyExists): services "order" already exists
이미 해당 객체가 존재한다는 오류. 해당 객체를 삭제:

kubectl delete <svc | deploy> <objectid>
상세설명
