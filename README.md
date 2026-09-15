## GitHub Actions 기반 ASG 컨테이너 자동 배포 구성

이번 실습에서는 GitHub Actions, Amazon ECR, Amazon S3, AWS Systems Manager(SSM), Application Load Balancer(ALB), Auto Scaling Group(ASG)을 연계하여 Nginx와 FastAPI 컨테이너를 자동으로 배포하는 환경을 구성하였다.

먼저 Amazon Linux 2023 기반 EC2 인스턴스에 Docker와 Docker Compose를 설치한 뒤 AMI를 생성하였다. 해당 AMI를 사용하는 Launch Template을 생성하고, EC2 인스턴스가 S3, ECR, SSM을 사용할 수 있도록 IAM Instance Profile에 `AmazonS3ReadOnlyAccess`, `AmazonSSMManagedInstanceCore`, `AmazonEC2ContainerRegistryReadOnly` 권한을 연결하였다.

Launch Template을 기반으로 Auto Scaling Group을 구성하고 Target Group 및 ALB와 연결하였다. ASG에서 생성되는 EC2 인스턴스에는 `Cicd=asg` 태그를 적용하여 GitHub Actions에서 SSM의 태그 기반 Target 기능을 이용해 여러 인스턴스를 동시에 배포 대상으로 지정할 수 있도록 구성하였다.

GitHub Actions가 실행되면 다음과 같은 순서로 CI/CD가 진행된다.

1. Nginx와 FastAPI Docker 이미지를 각각 빌드한다.
2. 빌드한 이미지를 Amazon ECR Repository에 Push한다.
3. `docker-compose.yaml`과 Nginx 정적 HTML 파일을 Amazon S3에 업로드한다.
4. SSM `send-command`를 이용하여 `Cicd=asg` 태그가 지정된 ASG 인스턴스에 배포 명령을 전달한다.
5. 각 EC2 인스턴스는 S3에서 최신 `docker-compose.yaml`과 HTML 파일을 다운로드한다.
6. ECR에 로그인한 후 최신 Nginx와 FastAPI 이미지를 Pull한다.
7. `docker compose up -d --remove-orphans` 명령으로 컨테이너를 실행 및 갱신한다.

### Nginx와 FastAPI 연결 구조

Nginx와 FastAPI는 Docker Compose에서 동일한 Docker Network에 연결하였다.

```text
Client
  ↓
ALB
  ↓
ASG EC2
  ↓
Nginx Container :80
  ├── /        → 정적 HTML
  │
  └── /api/*   → FastAPI Container :8000
````

Nginx의 `/api/` 요청은 Docker Network 내부의 FastAPI 서비스 이름을 이용하여 전달되도록 Reverse Proxy를 구성하였다. 따라서 FastAPI의 8000번 포트를 EC2 외부에 직접 공개하지 않고 Nginx를 통해서만 접근하도록 구성하였다.

정적 HTML 파일은 S3에 저장하고, 배포 시 각 EC2 인스턴스의 `/nginx/html` 경로로 동기화하였다. Docker Compose에서는 해당 디렉터리를 Nginx 컨테이너의 `/usr/share/nginx/html`에 Bind Mount하여 S3에서 전달된 최신 HTML 파일을 Nginx가 서비스하도록 구성하였다.

```text
S3 html/
   ↓ aws s3 sync
EC2 /nginx/html
   ↓ Bind Mount
Nginx /usr/share/nginx/html
```

### ASG 인스턴스 자동 복구

GitHub Actions를 이용한 배포뿐만 아니라 ASG에서 기존 EC2 인스턴스가 삭제되고 새로운 인스턴스가 생성되는 경우에도 별도의 수동 작업 없이 서비스를 복구할 수 있도록 Launch Template의 User Data를 구성하였다.

새로운 EC2 인스턴스가 생성되면 User Data에서 다음 작업을 자동으로 수행한다.

```text
새 EC2 생성
   ↓
Docker 서비스 시작
   ↓
S3에서 HTML 및 docker-compose.yaml 다운로드
   ↓
ECR 로그인
   ↓
Docker 이미지 Pull
   ↓
docker compose up
   ↓
Nginx + FastAPI 실행
   ↓
Target Group Health Check
   ↓
ALB 트래픽 수신
```

따라서 ASG의 기존 인스턴스를 삭제하더라도 ASG가 새로운 인스턴스를 자동으로 생성하고, 해당 인스턴스는 S3와 ECR에 저장된 최신 배포 파일을 이용하여 기존과 동일한 서비스 상태를 자동으로 복구한다.

### 최종 배포 구조

```text
GitHub Push
     ↓
GitHub Actions
     │
     ├── Nginx Image ───→ ECR
     │
     ├── FastAPI Image ─→ ECR
     │
     ├── HTML ──────────→ S3
     │
     └── docker-compose → S3
                         │
                         ↓
                SSM Tag Target
                 Cicd=asg
                         ↓
                  ASG EC2 Instances
                         ↓
                  Docker Compose
                   ↙           ↘
             Nginx              FastAPI
              :80                :8000
                ↑
                │
               ALB
```

최종적으로 ALB DNS 주소의 `/` 경로를 통해 Nginx 정적 페이지를 확인하고, `/api/` 경로를 통해 Nginx Reverse Proxy를 거쳐 FastAPI가 정상적으로 응답하는 것을 확인하였다. 또한 ASG 인스턴스를 삭제한 뒤 새로운 인스턴스가 자동 생성되어 별도의 GitHub Actions 재실행 없이 동일한 서비스가 정상적으로 복구되는 것까지 확인하였다.

```

이 정도면 단순히 “뭘 했다” 수준이 아니라, 나중에 네가 다시 봐도 **GitHub Actions는 기존 인스턴스에 새 버전을 배포하고, Launch Template User Data는 새 ASG 인스턴스를 자동 복구한다**는 핵심까지 바로 떠올릴 수 있는 README가 돼.
```
