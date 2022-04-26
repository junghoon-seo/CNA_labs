Instruction
Istio Circuit Breaker
Istio Circuit Breaker 기능 중, 장애가 감지된 서비스를 서비스 대상에서 일정시간 동안 제외(Pool Ejection)하는 Service Resiliency 를 실습한다.
Delivery 이미지 배포
배송 마이크로서비스를 배포한다.
kubectl create deploy delivery --image=ghcr.io/acmexii/delivery:istio-v1 -n tutorial
kubectl expose deploy delivery --port=8080 -n tutorial
Circuit Breaker 설치
kubectl apply -f - << EOF
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: dr-delivery
    namespace: tutorial
  spec:
    host: delivery
    trafficPolicy:
      outlierDetection:
        consecutive5xxErrors: 1
        interval: 1s
        baseEjectionTime: 3m
        maxEjectionPercent: 100
EOF
위 CB정책은 delivery 서비스의 라우팅 대상 컨테이너 목록에서 1초단위로 분석하여 1번이라도 서버 오류가 발생 시, 3분동안 라우팅에서 제외하며, 모든 컨테이너가 제외될 수 있음을 나타낸다.
모든 업스트림 서비스가 장애인 경우, 이스티오는 ‘No healthy upstream.’ 오류를 리턴한다.
Circuit Breaker 테스트 환경구성
배송서비스의 Replica를 3개로 스케일아웃한다.
설치된 Http Client 컨테이너에 접속한다.
kubectl scale deploy delivery --replicas=3 -n tutorial
kubectl exec -it pod/[SIEGE POD객체] -n tutorial -c siege-nginx  -- /bin/bash
Circuit Breaker 동작 확인
배송서비스 확인 (Replica가 3개 이므로, 아래 명령을 3번 이상 호출하여 호스트 정보 확인)
http http://delivery:8080/actuator/echo
아래 '/actuator/down’을 호출하면, 3개의 컨테이너 중 해당 호출을 받은 컨테이너는 ‘서비스 다운’ 상태를 가지게 된다. (1회만 호출)
http PUT http://delivery:8080/actuator/down
이 상태에서 아래 '/actuator/health’를 호출하면, ‘서비스 다운’ 상태인 컨테이너는 5xx 오류를 응답하게 된다. (3회 호출)
http GET http://delivery:8080/actuator/health
이 때, 서킷브레이커가 발동하여 해당 컨테이너를 (CB설정에 따라) 3분동안 Pool에서 Ejection 한다.
가용 서비스를 확인해 보면, 아래 명령에 대해 컨테이너 2개만 응답하는 것이 확인된다. (3회 이상 반복 실행)
http http://delivery:8080/actuator/echo
Pool Ejection 타임(3’) 경과후엔 배송서비스가 다시 3개가 동작됨이 확인된다.
http http://delivery:8080/actuator/echo
Kiali를 통한 Circuit Breaker 확인
모니터링 시스템(Kiali)에 접속하여 (http://Kiali EXTERNAL-IP:20001)배송서비스 Circuit Breaker 배지(Badge) 확인
Kiali 가 LoadBalancer로 노출되어있지 않다면

kubectl patch svc kiali -n istio-system -p '{"spec": {"type": "LoadBalancer"}}'
