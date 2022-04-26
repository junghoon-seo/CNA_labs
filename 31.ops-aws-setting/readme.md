Instruction
아마존 IAM 체계
AWS IAM 체계는 다음의 AWS ID 와 사용자 이름, 사용자 액세스 키, 스크릿 키 같은 여러가지 ID와 패스워드들을 가지고 있다.

AWS ID: 숫자로 채번된 사용자의 ID
AWS Access Key ID: AWS CLI 를 사용하기 위해서 제공되는 Access key
AWS Secret Acess Key: 위의 Acess Key 에 대한 패스워드격
왜 이렇게 가능한 복잡하게 만들어놨는지는 제프베조스에게 물어볼

아마존 콘솔에서 키들을 얻는 방법
콘솔접속 후 >
상단의 서비스 검색창에서 “IAM” 으로 검색 >
IAM (AWS 리소스에 대한 액세스 관리) 메뉴 진입 >
“사용자: XX” 클릭 >
사용자 목록에서 내 user id 를 선택 >
탭에서 “보안 자격 증명”을 선택 >
액세스 키 만들기 >
csv 파일 다운로드
AWS 콘솔 로그인
로그인 URL: 강의중에 알려드릴게요
사용자 이름: user01 ~ 22
패스워드: 강의중에 알려드릴게요 ^^
AWS CLI 환경 설정
aws configure

AWS Access Key ID: [AWS 액세스 키]
AWS Secret Access Key: [시크릿 키]
Default region name : [본인의 리젼]
Default output format : json
리젼은

(오레곤)us-west-2 --> User 1 ~ 5
(오사카)ap-northeast-3 - user 6 ~ 10
(런던)eu-west-2 - user 11 ~ 15
(싱가포르)ap-southeast-1 - user 16 ~ 20
(오하이오)us-east-2 - 21 ~ 25
(버지니아 북부)us-east-1 - user 26 ~ 30

클러스터 생성
eksctl create cluster --name [mycluster-userid] --version 1.19 --spot --managed --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3
생성된 클러스터의 API-Server 주소를 나중에 코드빌드에 설정해주어야 하기 때문에 생성된 후에 다시 확인해야 한다.
(약 15분이 소요되기 때문에 그 사이에 다음과정을 진행하셔도 됩니다)

AWS 콘솔에 로그인하여 EKS 서비스가 잘 생성되었는지 확인한다.

클러스터 제거할땐:
eksctl delete cluster [mycluster]
이름을 까먹었을때
eksctl get clusters

클러스터에 접속하기 위한 설정 다운로드
aws eks --region ap-northeast-2 update-kubeconfig --name [Cluster Name]
접속이 정상적으로 되었다면:

kubectl get nodes
의 결과가 잘 나오면 됩니다.

ECR 사용하기
ECR 에 docker 명령을 로그인시키기 위해서 먼저 docker password 를 얻어온다:

aws --region "리전명" ecr get-login-password 

긴 암호화된 패스워드 문자열이 출력됨
위의 문자열을 docker login 명령과 함께 -p 옵션으로 붙여넣기 한다. 이때 registry 서버 주소도 같이 넘겨준다:

docker login --username AWS -p 아주긴도커패스워드 [AWS유저아이디-숫자로만된].dkr.ecr.[리전명].amazonaws.com
docker login --username AWS -p 아주긴도커패스워드 979050235289.dkr.ecr.ap-northeast-2.amazonaws.com
잘 로그인되었다면, 앞서 예제에서 다루었던 order 마이크로 서비스를 push 해보자

처음으로 ECR에 이미지를 올리는 경우 다음과 같이 해당 산출물의 Repo 를 먼저 만든 후 진행합니다:

AWS console 접속 >
서비스 검색 “ECR”로 검색 >
왼쪽메뉴의 “Repositories” 를 선택 >
리포지토리 생성 클릭
Repository 명의 끝 부분에 “order” 입력
리포지토리 생성
git clone https://github.com/event-storming/monolith

cd monolith
mvn package -B
docker build -t [AWS유저아이디-숫자로만된].dkr.ecr.ap-northeast-2.amazonaws.com/order:v1 .
docker push [AWS유저아이디-숫자로만된].dkr.ecr.ap-northeast-2.amazonaws.com/order:v1
push 가 잘되었다면 ECR 의 웹UI 에 이미지가 표시될 것이며, 다음명령으로 Deploy 가 잘 되는지 확인한다:

kubectl create deploy order --image=[AWS유저아이디-숫자로만된].dkr.ecr.ap-northeast-2.amazonaws.com/order:v1
다음과 같이 po 를 확인한다:

> kubectl get po

NAME                     READY   STATUS    RESTARTS   AGE
order-7d66c76dcd-j4nbq   1/1     Running   0          18s
다음과 같이 log 를 확인한다:

> kubectl logs -f order-7d66c76dcd-j4nbq
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.1.RELEASE)
 
 Hibernate: 
    insert 
    into
        ProductOption
        (id, description, name, optionName, PRODUCT_ID) 
    values
        (null, ?, ?, ?, ?)
Hibernate: 
    insert 
    into
        ProductOption
        (id, description, name, optionName, PRODUCT_ID) 
    values
        (null, ?, ?, ?, ?)
 ...
