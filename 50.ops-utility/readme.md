Instruction
Helm
Helm(패키지 인스톨러) 설치

Helm 3.x 설치(권장)
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
Kafka
helm repo add incubator https://charts.helm.sh/incubator 
helm repo update 
kubectl create ns kafka 
helm install my-kafka --namespace kafka incubator/kafka 
혹은

helm repo update
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create ns kafka
helm install my-kafka bitnami/kafka --namespace kafka
Kafka 내부에 진입하여 메시지 확인하기

kubectl run my-kafka-client --restart='Never' --image docker.io/bitnami/kafka:2.8.0-debian-10-r0 --namespace kafka --command -- sleep infinity
    kubectl exec --tty -i my-kafka-client --namespace kafka -- bash

    PRODUCER:
        kafka-console-producer.sh \
            --broker-list my-kafka-0.my-kafka-headless.kafka.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \
            --bootstrap-server my-kafka.kafka.svc.cluster.local:9092 \
            --topic test \
            --from-beginning

HTTPie Pod
cat <<EOF | kubectl apply -f -
apiVersion: "v1"
kind: "Pod"
metadata: 
  name: httpie
  labels: 
    name: httpie
spec: 
  containers: 
    - 
      name: httpie
      image: clue/httpie
      command:
        - sleep
        - "36000"
EOF
생성후, 접속:

kubectl exec -it httpie bin/bash
Seige Pod
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
생성후, 접속:

kubectl exec -it siege bin/bash
