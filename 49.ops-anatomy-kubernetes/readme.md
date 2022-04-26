Instruction
Kubernetes Installation
Preparing a VM & Connect
Prepare any Cloud VM(EC2, Agent, Compute Engine)
아마존 콘솔 접속
서비스 검색 > “EC2” 선택
실행중인스턴스 > “인스턴스 시작”
Ubuntu Server 20.04 LTS 선택
t2.micro 선택
검토 및 시작
새 키 페어 생성 “ubuntu” > 다운로드
인스턴스 시작
보안 그룹명이 충돌나는 경우 임의의 보안그룹 명으로 변경합니다.

인스턴스 목록 > 생성한 인스턴스 선택 > 연결

받은 pem 파일을 개발 환경에 업로드 : File Menu > UploadFiles…

chmod 400 ubuntu.pem

Connect to VM (here, aws EC2 Linux-AMI)

ssh -i ./k8s.pem ubuntu@ec2-3-35-209-101.ap-northeast-2.compute.amazonaws.com
Set root passwd and switching User to root

sudo passwd root
su -
Update and upgrade Ubuntu:

apt update
apt upgrade -y
Install K8s Binaries
The next steps will prepare CRI and Software for kube setup.

도커엔진을 설치:

apt install docker.io -y
K8s 의 바이너리 파일을 다운 받는다:

wget https://storage.googleapis.com/kubernetes-release/release/v1.9.11/kubernetes-server-linux-amd64.tar.gz
tar -xzf kubernetes-server-linux-amd64.tar.gz
Install the K8s binaries:

cd kubernetes/server/bin/
mv kubectl kubelet kube-apiserver kube-controller-manager kube-scheduler kube-proxy /usr/bin/
cd
kubectl 이 실행되는지 확인한다:

kubectl
압축파일이 더 이상 필요없으니 삭제:

rm -rf kubernetes kubernetes-server-linux-amd64.tar.gz
Creating the Kube
Set up kubelet
다음단계는 Kubelet 의 동작을 이해한다

큐블릿을 위한 디렉토리를 만든다:

mkdir -p /etc/kubernetes/manifests
큐블릿을 백그라운드(데몬)로 실행한다:

kubelet --pod-manifest-path /etc/kubernetes/manifests &> /etc/kubernetes/kubelet.log &
큐블릿의 상태와 기본 로그를 확인한다:

ps -au | grep kubelet
head /etc/kubernetes/kubelet.log
큐블릿의 manifest directory 에 yaml 을 위치하면, 큐블릿이 해당 yaml 을 도커엔진을 통해 컨테이너를 생성해주기 때문에 그를 테스트할 yaml 을 하나 만든다.

cat <<EOF > /etc/kubernetes/manifests/kubelet-test.yaml
apiVersion: v1
kind: Pod
metadata:
name: kubelet-test
spec:
containers:
- name: alpine
    image: alpine
    command: ["/bin/sh", "-c"]
    args: ["while true; do echo SuPeRgIaNt; sleep 15; done"]
EOF
파일이 생성된 직후에 도커에 해당 pod 가 생성되었는지 확인한다.:

docker ps
컨테이너의 로그를 확인한다

docker logs {CONTAINER ID}
Set up etcd
이번단계는 etcd 에 쿠버네티스의 상태를 보존하는 것을 확인한다.

etcd 와 etcdctl를 다운받고 압축해제 한다:

wget https://github.com/etcd-io/etcd/releases/download/v3.2.26/etcd-v3.2.26-linux-amd64.tar.gz
tar -xzf etcd-v3.2.26-linux-amd64.tar.gz
etcd 와 etcdctl 바이너리를 설치한다.

mv etcd-v3.2.26-linux-amd64/etcd /usr/bin/etcd
mv etcd-v3.2.26-linux-amd64/etcdctl /usr/bin/etcdctl
불필요한 리소스를 삭제한다:

rm -rf etcd-v3.2.26-linux-amd64 etcd-v3.2.26-linux-amd64.tar.gz
etcd를 실행한다:

etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://localhost:2379 &> /etc/kubernetes/etcd.log &
etcd 가 건강하게 실행되고 있는지 다음과 같이 확인된다::

etcdctl cluster-health
쿠버네티스 내에 디플로이된 모든 리소스들을 한번 확인해본다:

kubectl get all --all-namespaces
Set up kube-apiserver
이번 단계는 apiserver 가 어떻게 실행되는지 이해합니다.

kube-apiserver 를 실행한다

kube-apiserver --etcd-servers=http://localhost:2379 --service-cluster-ip-range=10.0.0.0/16 --bind-address=0.0.0.0 --insecure-bind-address=0.0.0.0 &> /etc/kubernetes/apiserver.log &
실행상태와 시작로그를 확인한다:

ps -au | grep apiserver
head /etc/kubernetes/apiserver.log
API 를 확인해보고 반응을 확인한다:

curl http://localhost:8080/api/v1/nodes
kubeconfig File 을 설정하여 kubectl 설정
이번 과정은 kubectl 을 제대로 설정해본다.

kubectl 이 API server 를 제대로 바라보고 있는지 확인한다

kubectl cluster-info
kubeconfig file에 API server address 가 잘 설정되었는지 확인한다

kubectl config set-cluster kube-from-scratch --server=http://localhost:8080
kubectl config view
우리가 접속할 apiserver 로 kubectl 의 context 를 지정한다:

kubectl config set-context kube-from-scratch --cluster=kube-from-scratch
kubectl config view
Use the context created earlier for kubectl:

kubectl config use-context kube-from-scratch
kubectl config view
check that resources can now be seen on the cluster:

kubectl get all --all-namespaces
kubectl get node
Set up the New Config for kubelet
The next steps will take the configuration created and use it to configure kubelet.

Restart kubelet with a new flag pointing it to the apiserver (this step may fail once or twice, try again):

pkill -f kubelet
kubelet --register-node --kubeconfig=".kube/config" &> /etc/kubernetes/kubelet.log &
Check its status and initial logs:

ps -au | grep kubelet
head /etc/kubernetes/kubelet.log
Check to see that kubelet has registered as a node:

kubectl get node
Check to see the old Pod is not coming up:

docker ps
Check that the Pod manifest is still present:

ls /etc/kubernetes/manifests
이제 kubelet 으로 pod 를 생성해봅니다. 우리가 설치한 control plane 들이 잘 동작하는지 확인하게 됩니다:

cat <<EOF > ./kube-test.yaml
apiVersion: v1
kind: Pod
metadata:
name: kube-test
labels:
    app: kube-test
spec:
containers:
- name: nginx
    image: nginx
    ports:
    - name:  http
    containerPort: 80
    protocol: TCP
EOF
kubectl create -f kube-test.yaml
Check the Pod’s status:

kubectl get po
Set up kube-scheduler
다음 단계는 scheduler 를 통해서 pod 가 생성될 수 있도록 설정합니다

Scheduler를 시작합니다:

kube-scheduler --master=http://localhost:8080/ &> /etc/kubernetes/scheduler.log &
스케쥴러의 상태와 시작로그를 확인합니다:

ps -au | grep scheduler
head /etc/kubernetes/scheduler.log
Check to see if the Pod was scheduled:

kubectl get po
Delete the Pod:

kubectl delete po --all
Set up kube-controller-manager
다음단계는 ReplicaSet 등을 관리해주는 다양한 쿠버네티스 객체의 행위를 지원하는 controller 매니저를 설치합니다.

레플리카 3개가 설정된 Deployment를 준비한다:

cat <<EOF > ./replica-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: replica-test
spec:
replicas: 3
selector:
    matchLabels:
    app: replica-test
template:
    metadata:
    name: replica-test
    labels:
        app: replica-test
    spec:
    containers:
    - name: nginx
        image: nginx
        ports:
        - name:  http
        containerPort: 80
        protocol: TCP
EOF
kubectl create -f replica-test.yaml
Check the Deployment’s status:

kubectl get deploy
Pending 상태임을 확인한다:

kubectl get po
controller-manager를 시작한다:

kube-controller-manager --master=http://localhost:8080 &> /etc/kubernetes/controller-manager.log &
Check its status and initial logs:

ps -au | grep controller
head /etc/kubernetes/controller-manager.log
Check the status of the Deployment:

kubectl get deploy
이제 3개의 Replica 가 생성되어 각 pod 의 상태가 AVAILABLE 로 변경된 것을 확인한다:

kubectl rollout resume deploy/replica-test
kubectl rollout status deploy/replica-test
Check the new Pods:

kubectl get po
Set up kube-proxy
이제 외부 트래픽을 Pod 로 전송하는 kube-proxy 를 설치하고 이들이 어떻게 동작하는지 이해한다:

앞서의 replica-test Deployment에 대한 service 를 만들어준다:

cat <<EOF > ./service-test.yaml
apiVersion: v1
kind: Service
metadata:
name: replica-test
spec:
type: ClusterIP
ports:
- name: http
    port: 80
selector:
    app: replica-test
    EOF
    kubectl create -f service-test.yaml
Curl the service to see if any Pod is contacted:

kubectl get svc
curl {CLUSTER IP}:80
Start kube-proxy:

kube-proxy --master=http://localhost:8080/ &> /etc/kubernetes/proxy.log &
Check its status and initial logs:

ps -au | grep proxy
head /etc/kubernetes/proxy.log
Curl the Service again to see if any Pod is contacted:

kubectl get svc
curl {CLUSTER IP}:80
