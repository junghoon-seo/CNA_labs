Instruction
Virtual Host based Ingress Example
apiVersion: "extensions/v1beta1"
kind: "Ingress"
metadata: 
  name: "istio-ingress"
  namespace: "istio-system"
  annotations: 
    kubernetes.io/ingress.class: "nginx"
spec: 
  rules: 
    - host: "kiali.service.com"
      http: 
        paths: 
          - 
            path: /
            pathType: Prefix
            backend: 
              serviceName: kiali
              servicePort: 20001

    - host: "prom.service.com"
      http: 
        paths: 
          - 
            path: /
            pathType: Prefix
            backend: 
              serviceName: prometheus
              servicePort: 9090

    - host: "gra.service.com"
      http: 
        paths: 
          - 
            path: /
            pathType: Prefix
            backend: 
              serviceName: grafana
              servicePort: 3000

을 ingress.yaml 파일로 만들어 저장한후 생성한다

$ kubectl create -f ingress.yaml

이때 yaml 문법 validation 오류가 생긴다면, 설치된 쿠버네티스의 버전에 따라 발생할 수 있으며, validate 옵션을 해제하여 설정한다:

생성된 ingress 의 상태를 확인한다:

$ kubectl get ingress -n istio-system -w

NAME               HOSTS   ADDRESS                                                                        PORTS   AGE
istio-ingress   *       ???   80      7m36s
아무리 기다려도 ADDRESS 부분에 값이 채워지지 않음을 알 수 있다. 원인은 내게 gateway provider 가 없기 때문이다. Ingress 는 Kubernetes 의 스펙일 뿐, 이를 실질적으로 지원하는 ingress controller 가 필요하기 때문이다. 다행히, 우리에겐 무료로 사용할 수 있는 nginx 인그레스 프로바이더를 사용할 수 있다.

Ingress Provider 설치하기
오픈소스 ingress provider 인 nginx ingress controller 를 설치하기 위해서는 하나 이상의 kubernetes 구성요소들을 설치해야 하기 때문에 이를 쉽게 Helm Chart 를 통해서 설치할 수 있다.

Helm으로 Ingress Controller 설치
Helm repo 설정
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create namespace ingress-basic
nginx controller 설치
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace=ingress-basic
설치확인
Ingress Controller의 EXTERNAL-IP가
API Gateway 엔드포인트: 메모 必
kubectl get all --namespace=ingress-basic
이제, 자동으로 ingress 의 ADDRESS 부분의 설정이 채워지는 것을 확인한다:
$ kubectl get ingress
NAME               HOSTS   ADDRESS                                                                        PORTS   AGE
istio-ingress   *       acbdde7c8e29f451daee5605b8c7840c-1087513605.ap-northeast-2.elb.amazonaws.com   80      7m36s
아마존의 경우 획득한 주소가 도메인 네임이므로, IP Address 를 얻기위해서 ping 을 이용하여 address 를 얻는다:

ping acbdde7c8e29f451daee5605b8c7840c-1087513605.ap-northeast-2.elb.amazonaws.com

#  리턴되는 ip address 를 획득
가상호스트를 테스트하기 위해서 내의 hosts 파일에 아래를 추가한다:
(윈도우에서는 C:\Windows\System32\drivers\etc\hosts 에서 찾을 수 있고 리눅스와 맥은 /etc/hosts 파일을 수정하면 된다)

<획득한 ingress의 External IP>  kiali.service.com, prom.service.com, gra.service.com
파일 저장을 위하여 윈도우에서 메모장으로 열때 “관리자 권한으로 실행” 하여 메모장 애플리케이션을 열어야 하고, 리눅스와 맥에서는 “sudo vi /etc/hosts” 로 수정해야 한다.

이제 브라우저를 열고 prom.service.com와 gra.service.com 에 접속해본다.

혹은 curl로 확인가능하다:

curl -H "Host: prom.service.com" <IP 주소>

<a href="/graph">Found</a>.
