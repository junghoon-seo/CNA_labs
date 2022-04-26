Instruction
Auto Scale-Out 실습 (hpa: HorizontalPodAutoscaler 설정)
클라우드의 리소스를 잘 활용하기 위해서는 요청이 적을때는 최소한의 Pod 를 유지한 후에 요청이 많아질 경우 Pod를 확장하여 요청을 처리할 수 있다.
Pod 를 Kubernetes에서 수평적으로 확장하는 방법을 HorizontalPodAutoscaler(HPA) 라고 부른다. replicas 를 관리하는 Deployment, StatefulSet 에 적용이 가능하고, 확장이 불가능한 DaemonSets 에는 설정이 불가능하다.

HPA는 워크로드의 CPU 또는 메모리를 측정하여 작동하기 때문에 Kubernetes 에 metric server 를 필수적으로 설치가 되어있어야 한다.

이번시간에는 HPA 설정을 적용 한 후에, siege 라는 부하 테스트 툴을 사용하여 서비스에 부하를 주어 Pod 가 Auto Scale-Out 되는 실습을 한다.

선행과정
이전 랩인 [운영] 애플리케이션의 패키징,도커라이징,클러스터 배포를 실행하여 클러스터에 order 서비스가 배포가 되어있어야 한다.

kubectl get svc 하였을때 order 서비스 존재확인.
kubectl get pod 하였을때 order의 STATUS 가 Running 상태확인.
부하 테스트 Pod 설치

아래 스크립트를 terminal 에 복사하여 siege 라는 Pod 를 생성한다.
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
생성된 siege Pod 안쪽에서 정상작동 확인
kubectl exec -it siege -- /bin/bash
siege -c1 -t2S -v http://order:8080/orders
exit
metric server 설치 확인 방법

kubectl top pods 를 하였을때 아래와 같이 정보가 나오면 설치가 되어있다.
NAME                     CPU(cores)   MEMORY(bytes)   
order-684647ccf9-ltlqg   3m           288Mi           
siege                    0m           8Mi   
metric server가 설치 안되어있다면 아래와 같은 명령어로 설치한다.
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
kubectl get deployment metrics-server -n kube-system

1. Auto Scale-Out 설정
1.0 Auto Scaler를 설정한다
오토 스케일링 설정명령어 호출

kubectl autoscale deployment order --cpu-percent=20 --min=1 --max=3
“cpu-percent=50 : Pod 들의 요청 대비 평균 CPU 사용율 (여기서는 요청이 200 milli-cores이므로, 모든 Pod의 평균 CPU 사용율이 100 milli-cores(50%)를 넘게되면 HPA 발생)”

kubectl get hpa 명령어로 설정값을 확인 한다.

NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
order   Deployment/order   <unknown>/20%   1         3         0          7s
1.1 배포파일에 CPU 요청에 대한 값을 지정한다.

shopmall > order > kubernetes 폴더로 이동하여 deployment.yaml 파일을 수정한다.
19 Line 의 image 을 jinyoung/monolith-order:v20210602
로 변경한다.
21과 22 Line의 ports 와 readinessProbe 사이에 resources.requests.cpu: "250m"을 추가한다.
indent 를 주의해야 한다
파일을 저장한다.
		ports:
          - containerPort: 8080
        resources:
          requests:
            cpu: "200m"
        readinessProbe:
1.2 터미널을 열어서 변경된 yaml 파일을 사용하여 쿠버네티스에 배포한다.

cd shopmall/order/kubernetes
kubectl apply -f deployment.yml
1.3 배포 완료 후 kubectl get deploy order -o yaml 명령을 쳐서 image 와 resources의 값이 정상적으로 설정되어있는지 확인

kubectl get po 실행하여 STATUS가 정상적으로 Running 상태 확인
2. Auto Scale-Out 증명
2.1 새로운 터미널을 열어서 seige 명령으로 부하를 주어서 Pod 가 늘어나도록 한다.

kubectl exec -it siege -- /bin/bash
siege -c20 -t40S -v http://order:8080/orders
exit
2.2 터미널 1개는 kubectl get po -w 명령을 사용하여 pod 가 생성되는 것을 확인한다.

order-7b76557b8f-bgptv   1/1     Running   0          34m
siege                    1/1     Running   0          33m
order-7b76557b8f-7g9d6   0/1     Pending   0          0s
order-7b76557b8f-hmssb   0/1     Pending   0          0s
order-7b76557b8f-7g9d6   0/1     ContainerCreating   0          0s
order-7b76557b8f-hmssb   0/1     ContainerCreating   0          0s
order-7b76557b8f-7g9d6   0/1     Running             0          6s
order-7b76557b8f-hmssb   0/1     Running             0          6s
order-7b76557b8f-7g9d6   1/1     Running             0          23s
order-7b76557b8f-hmssb   1/1     Running             0          27s
2.3 kubectl get hpa 명령어로 CPU 값이 늘어난 것을 확인 한다.

NAME    REFERENCE          TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
order   Deployment/order   1152%/20%   1         3         3          37m
