Instruction
Argo CD 를 통한 배포
Argo CD 의 홈페이지를 방문한다:
https://argo-cd.readthedocs.io/en/stable/

Getting Started 메뉴를 접속하여, argo cd 를 설치한다:

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
Argo CD UI 를 접속하기 위하여 LoadBalancer 로 전환한다:

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

Argo CD UI 의 External IP 주소를 획득한다

kubectl get svc argocd-server -n argocd
접속한다.

Argo CD 는 기본 https 로 UI 서비스가 열리므로, 인증서가 없이 서비스를 열었으므로, 이를 그냥 접속하기 위해서 해당 페이지에서 허공에 대고 “thisisunsafe” 를 입력하면 다음과 같은 페이지로 넘어간다 ㅡㅡ;



접속 user id 는 admin 이고 password 는 다음과 같이 Secret 에서 얻어내어야 한다 (무슨 CD 툴이 왠 보안에 엄청 신경을):

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
왼쪽 메뉴에서 New App 을 클릭하여 Git 주소가 포함된 Application 정보를 등록한다:



Guestbook Application 을 등록한다:



https://github.com/argoproj/argocd-example-apps.git 를 접속한 후, 이를 Fork 한다.

내 계정으로 복제된 guest book application 의 git 주소를 argo 에 등록한다.



배포될 타겟 클러스터를 지정한다



kubernetes.default.svc 가 내가 포함된 서비스의 기본 접속 주소이다.
namespace 를 “guestbook” 으로 줘본다.

git 에 변화를 주고, 이를 동기화 시켜서 반영이 되는지 확인한다:



