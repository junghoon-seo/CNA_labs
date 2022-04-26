Instruction
셀프힐링 실습 (livenessProbe 설정)
Pod는 정상적으로 작동하지만 내부의 어플리케이션이 반응이 없다면, 컨테이너는 의미가 없다.
위와 같은 경우는 어플리케이션의 Deadlock 또는 메모리 과부화로 인해 발생할 수 있으며, 발생했을 경우 컨테이너를 다시 시작해야 한다.
Liveness probe는 Pod의 상태를 체크하다가, Pod의 상태가 비정상인 경우 kubelet을 통해서 재시작한다.

이번시간에는 특정 API 를 호출시 어플리케이션의 메모리 과부화를 발생시켜 서비스가 동작안하는 상황을 만든다.
그 후 livenessProbe 설정에 의하여 자동으로 서비스가 재시작 되는 실습을 한다.

선행과정
메모리 릭을 유도하는 소스코드의 확인: shopmall/order/src/main/java/shopmall/OrderController.java

mvn 패키징을 통해서 애플리케이션을jar파일로 압축한다:

 mvn package -B
애플리케이션을 도커허브나 플랫폼별 컨테이너 레지스트리에 order:memleak의 이름으로 push 한다
e.g. ACR case:
az acr build --registry user27 --image user27.azurecr.io/order:memleak .
- shopmall > order > kubernetes 폴더로 이동하여 deployment.yaml 파일을 수정한다.
- 19 Line 의 이미지명 부분을 생성한 이미지명으로 변경한다.
- cd /home/project/ops-deploy-my-app/shopmall/order/kubernetes
- kubectl apply -f deployment.yml
- kubectl apply -f service.yaml

- kubectl get po 실행하여 STATUS가 정상적으로 Running 상태 확인
1. livenessProbe 확인
1.1 터미널을 열어서 siege Pod 로 들어간 후 http 명령으로 주문서비스를 호출하여 정상작동하는지 확인한다.

kubectl exec -it siege -- /bin/bash
http http://order:8080/orders	
1.2 새로운 터미널을 열어서 Pod 의 변화를 살펴보기 위하여 watch 를 걸어놓는다

kubectl get po -w
1.3 order 서비스의 OrderController.java 코드를 살펴보면 메모리를 강제로 부하시키는 코드를 호출한다. (callMemleak 호출)

http http://order:8080/callMemleak
1.4 watch 걸어놓은 pod 에서 pod 의 상태를 확인한다.

RESTARTS 카운트가 증가하는 것을 확인 한다.
root@labs--201094368:~# kubectl get po -w
NAME                     READY   STATUS    RESTARTS   AGE
order-684647ccf9-ltlqg   1/1     Running   0          80s
siege                    1/1     Running   0          2m9s
order-684647ccf9-ltlqg   0/1     OOMKilled   0          116s
order-684647ccf9-ltlqg   0/1     Running     1          119s
order-684647ccf9-ltlqg   1/1     Running     1          2m17s
