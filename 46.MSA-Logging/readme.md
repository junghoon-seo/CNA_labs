Instruction
마이크로서비스 통합 로깅
EFK(Elasticsearch, Fluentd, Kibana) 스텍을 클러스터에 설치하여 마이크로서비스 로그를 중앙에서 통합 모니터링한다.
로그 수집기를 Fluentd 대신 동일 회사(Treasure Data)가 제작한 High Performance의 경량화 버전인 Fluent Bit를 적용한다.
수집 데이터 저장소인 Elasticsearch를 기반으로 Kibana에서 시각화하여 통합 로깅한다.
ElasticSearch, Kibana 설치
Helm으로 ElasticSearch와 Kibana를 차례로 설치한다.
helm repo add elastic https://helm.elastic.co
helm repo update
kubectl create namespace elastic
helm install elasticsearch elastic/elasticsearch -n elastic
helm install kibana elastic/kibana -n elastic
설치확인
kubectl get all -n elastic
kubectl get pods --namespace=elastic -l app=elasticsearch-master -w
-동작확인 : ElasticSearch의 default index목록이 조회되는지 확인한다.

kubectl port-forward -n elastic svc/elasticsearch-master 9200
curl http://localhost:9200/_cat/indices
Fluent Bit 설치
helm chart 를 사용하지 않고 ConfigMap과 DaemonSet 을 확인하면서 설치한다.
설치를 위한 YAML 엔드포인트는 본 단락 마지막에 제공된다.

1. SA 생성
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: elastic
2. ClusterRole 생성
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-read
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  verbs: ["get", "list", "watch"]
3. Role 바인딩
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-read
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit-read
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: elastic
4. Fluent Bit ConfigMap 설정
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: elastic
  labels:
    k8s-app: fluent-bit
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     debug
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
        # Logging 파이프라인
    @INCLUDE input-kubernetes.conf
    @INCLUDE filter-kubernetes.conf
    @INCLUDE output-elasticsearch.conf

  input-kubernetes.conf: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*_kube-system_*.log
        # Path에서 수집되는 데이터 태깅
        Tag               kube.*
        Read_from_head    true
        Parser            cri
    [INPUT]
        Name              tail
        Tag               shop.*
        Path              /var/log/containers/*_shop_*.log
        Multiline         on
        Read_from_head    true
        Parser_Firstline  multiline_pattern

  filter-kubernetes.conf: |
    [FILTER]
        Name                kubernetes
        # 모든 태그에 대해 kubernetes Filtering 처리. (k8s 메타정보로 Log Enrichment)
        Match               *
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
    [FILTER]
        Name                  multiline
        Match                 shop.*
        multiline.key_content log
        multiline.parser      java

  output-elasticsearch.conf: |
    [OUTPUT]
        Name            es
        Match           kube.*
        Host            ${FLUENT_ELASTICSEARCH_HOST}
        Port            ${FLUENT_ELASTICSEARCH_PORT}
        # kubernetes Sys 로그의 Index Name 설정
        Index           fluent-k8s
        Type            flb_type
        Logstach_Format On
        Logstach_Prefix fluent-k8s
        Retry_Limit     False
    [OUTPUT]
        Name            es
        Match           shop.*
        Host            ${FLUENT_ELASTICSEARCH_HOST}
        Port            ${FLUENT_ELASTICSEARCH_PORT}
        # shop 네임스페이스 로그의 Index Name 설정
        Index           fluent-shop
        Type            flb_type
        Logstach_Format On
        Logstach_Prefix fluent-shop
        Retry_Limit     False

  parsers.conf: |
    [PARSER]
        Name cri
        Format regex
        Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

    [PARSER]
        Name multiline_pattern
        Format regex
        Regex   ^\[(?<timestamp>[0-9]{2,4}\-[0-9]{1,2}\-[0-9]{1,2} [0-9]{1,2}\:[0-9]{1,2}\:[0-9]{1,2})\] (?<message>.*)
        Time_Key    time
        Time_Format %Y-%m-%
5. Fluent Bit DaemonSet 생성
위 ConfigMap(fluent-bit-config)을 사용하는 DaemonSet을 배포한다.
위에서 생성한 ConfigMap을 볼륨마운트해 /fluent-bit/etc/ 위치에 생성된 5개의 conf 파일을 데몬셋이 사용한다.
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: elastic
	...
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit
        imagePullPolicy: Always
        volumeMounts:
        ...
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      volumes:
      ....
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      serviceAccountName: fluent-bit
Fluent Bit를 아래의 YAML 엔드포인트를 이용해 차례로 설치한다.
kubectl apply -f https://raw.githubusercontent.com/event-storming/elasticsearch/main/service-account.yaml
kubectl apply -f https://raw.githubusercontent.com/event-storming/elasticsearch/main/role.yaml
kubectl apply -f https://raw.githubusercontent.com/event-storming/elasticsearch/main/role-binding.yaml
kubectl apply -f https://raw.githubusercontent.com/event-storming/elasticsearch/main/configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/event-storming/elasticsearch/main/daemonset.yaml
Fluent Bit 동작확인
kubectl get all -n elastic
kubectl port-forward -n elastic svc/elasticsearch-master 9200
curl http://localhost:9200/_cat/indices
- index 목록 중, 'fluent-shop, fluent-k8s'로 시작되는 index가 존재하면 성공
대상 마이크로서비스(12st Mall) 배포
shop 네임스페이스를 생성하고, 주문과 배송 마이크로서비스를 배포한다.
kubectl create ns shop
kubectl apply -f https://raw.githubusercontent.com/acmexii/demo/master/edu/order-liveness.yaml -n shop
kubectl expose deploy order --port=8080 -n shop
kubectl apply -f https://raw.githubusercontent.com/acmexii/demo/master/edu/delivery-rediness-v1.yaml -n shop
kubectl expose deploy delivery --port=8080 -n shop
Kibana를 통한 12st Mall 서비스 로깅
kibana 서비스를 Port-forwarding 하거나, 서비스를 LoadBalancer Type으로 수정후 접속한다.
kubectl port-forward -n elastic deployment/kibana-kibana 5601
OR, 
kubectl edit svc/kibana-kibana -n elastic
1. Index 패턴 생성
Kibana 접속 후, Management > Stack Management를 선택한다.
image

Kibana > Index Patterns 화면의 Search 필드에 'fluent-shop*'을 입력하고 Time field 에 @timestamp 를 선택하여 수집된 데이터를 인덱싱한다.
image

2. 로그 조회
Analytics > Discover 를 눌러 조회페이지를 오픈한다.
image

‘Add filter’ 에서 'kubernetes.namespace.name is shop’으로 조건을 지정한다.
image

조회할 Date Range에 인덱싱된 shop 네임스페이스 data가 존재하면 아래처럼 로그가 나타난다.
image

3. 로그리게이션 (Log + Aggregation)
로그가 표시되는 영역의 컬럼을 선택하여 주문, 배송 서비스의 Stack trace를 확인한다.
우측 Selected fields에서 log_processed.log와 kubernetes.labels.app을 선택한다.
image
