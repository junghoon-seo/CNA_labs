Instruction
Ingress
Ingress 의 설정
주문, 상품, 배송 서비스를 분기하는 Ingress 를 생성한다:

apiVersion: "extensions/v1beta1"
kind: "Ingress"
metadata: 
  name: "shopping-ingress"
  annotations: 
    kubernetes.io/ingress.class: "nginx"
spec: 
  rules: 
    - 
      http: 
        paths: 
          - 
            path: /orders
            pathType: Prefix
            backend: 
              serviceName: order
              servicePort: 8080
          - 
            path: /deliveries
            pathType: Prefix
            backend: 
              serviceName: delivery
              servicePort: 8080
          - 
            path: /products
            pathType: Prefix
            backend: 
              serviceName: product
              servicePort: 8080
을 ingress.yaml 파일로 만들어 저장한후 생성한다

$ kubectl create -f ingress.yaml

이때 yaml 문법 validation 오류가 생긴다면, 설치된 쿠버네티스의 버전에 따라 발생할 수 있으며, validate 옵션을 해제하여 설정한다:

생성된 ingress 의 상태를 확인한다:

$ kubectl get ingress shopping-ingress -w

NAME               HOSTS   ADDRESS                                                                        PORTS   AGE
shopping-ingress   *       ???   80      7m36s
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
shopping-ingress   *       acbdde7c8e29f451daee5605b8c7840c-1087513605.ap-northeast-2.elb.amazonaws.com   80      7m36s
발급된 주소에 path 까지 포함하여 접속을 시도해본다:

# http a52f3e05efdb2439e845aed8379437b4-1576614801.ap-northeast-2.elb.amazonaws.com/orders
HTTP/1.1 503 Service Temporarily Unavailable
Connection: keep-alive
Content-Length: 190
Content-Type: text/html
Date: Tue, 11 May 2021 06:25:37 GMT

<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>nginx</center>
</body>
</html>
HTML 이 출력되는 것으로보아 nginx 까지는 무사히 연결된 것으로 보이나, 해당 주소 (orders)에 접속이 안되는 것을 확인할 수 있다. 이것은 backend 인 order service 를 디플로이 하지 않았기 때문이다.

order 서비스와 delivery 서비스를 잘 디플로이 해주면, 해당 path 들로 path-based routing 이 잘 이루어짐을 알 수 있다.

Virtual Host based Ingress Example
apiVersion: "extensions/v1beta1"
kind: "Ingress"
metadata: 
  name: "shopping-ingress"
  namespace: "istio-system"
  annotations: 
    kubernetes.io/ingress.class: "nginx"
spec: 
  rules: 
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

가상호스트를 테스트하기 위해서 C:\Windows\System32\drivers\etc 내의 hosts 파일에 아래를 추가한다:

<획득한 ingress의 External IP>  prom.service.com, gra.service.com
