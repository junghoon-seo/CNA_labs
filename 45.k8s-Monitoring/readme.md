Instruction
Kubernetes Monitoring
CNCF의 모니터링 스텍인 프로메테우스와 Grafana를 사용해 k8s와 클러스터에 배포된 리소스를 모니터링 한다.
Deploy Prometheus / Grafana Monitoring Stack on Kubernetes
Step 1: Clone kube-prometheus project
cd /home/project
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus
Step 2: Create monitoring namespace, CustomResourceDefinitions & operator pod
rometheus Operator는 Prometheus 기반 Kubernetes 모니터링 스택을 관리하기 위한 Kubernetes Operator 패턴을 구현

kubectl create -f manifests/setup
Step 3: Deploy Prometheus Monitoring Stack on Kubernetes
kubectl create -f manifests/
설처된 프로메테우스 Pod를 확인한다.
kubectl get pods -n monitoring
설치된 서비스 목록을 확인한다.
kubectl get svc -n monitoring
Step 4: Access Prometheus, Grafana, and Alertmanager dashboards
서비스를 통해 Prometheus와 Grafana, Alertmanager에 접속한다.
kubectl port-forward을 이용하거나,
Service expose type를 수정한다.

# Service expose type를 수정
kubectl edit service/grafana -n monitoring
vi 편집기로 실행정보가 오픈된다.

: 입력
%s/ClusterIP/LoadBalancer/g 입력후엔터
:wq! 입력후 엔터
grafana 서비스가 LoadBalancer 타입으로 수정된다.

kubectl get svc -n monitoring
생성된 External-IP:3000 을 복사하여웹브라우저에서 접속한다.
디폴드 로그인 정보(admin / admin)를 입력한다.
grafana
Kubernetes Monitoring
설치된 프로메테우스와 Grafana 스텍으로 쿠버네티스를 리소스를 모니터링한다.

Grafana 왼쪽 메뉴에서 돋보기를 클릭한다.

grafana
클릭 후, 나타나는 Default 폴더를 펼친 후 PreSet 목록 중 원하는 차트를 클릭한다.

클릭한 주제의 차트가 아래처럼 출력된다.

grafana
원하는 Chart는 Cluster에 저장할 수 있고, Prometheus Query(PromQL)를 익혀 우리 팀의 챠트를 구성할 수 있다.
Grafana 사이트에서 원하는 차트를 검색하여 그대로 인용도 가능하다. https://grafana.com/grafana/dashboards/
위 사이트에서 필터링하여 차트 ID를 획득하여 설치한 Grafana에서 Import한다.

Destroying / Tearing down Prometheus monitoring stack
/home/project/kube-prometheus 폴더 상에서 아래 명령어로 모든 스텍을 삭제한다.
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
