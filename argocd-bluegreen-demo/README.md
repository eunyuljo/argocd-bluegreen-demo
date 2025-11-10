# ArgoCD + Argo Rollouts Blue-Green 배포 데모

ArgoCD로 배포하고 Argo Rollouts로 Blue-Green 전략을 사용하는 완전한 데모입니다.

## 특징

- ✅ **빌드 불필요**: 공개 nginx 이미지 사용
- ✅ **ArgoCD GitOps**: Git 저장소 기반 배포
- ✅ **Blue-Green 전략**: 안전한 무중단 배포
- ✅ **시각적 확인**: Blue(파란색) ↔ Green(초록색)
- ✅ **간단한 구조**: Kubernetes 초보자도 이해 가능

## 사전 요구사항

### 필수
- Kubernetes 클러스터
- kubectl 설치 및 클러스터 접근 권한
- Git 저장소 (GitHub, GitLab 등)

### 설치해야 할 것
1. **ArgoCD**
2. **Argo Rollouts**

## 1단계: Argo Rollouts 설치

```bash
# Helm으로 설치
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argo-rollouts argo/argo-rollouts \
  -n argo-rollouts \
  --create-namespace

# 설치 확인
kubectl get pods -n argo-rollouts
```

## 2단계: ArgoCD 설치

```bash
# ArgoCD 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 설치 확인
kubectl get pods -n argocd -w

# ArgoCD 접속을 위한 Port Forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**ArgoCD 접속**: https://localhost:8080
- Username: `admin`
- Password: (위에서 확인한 비밀번호)

## 3단계: Git 저장소 준비

### 옵션 A: 기존 저장소 Fork (추천)

1. https://github.com/argoproj/argo-helm 접속
2. 우측 상단 **Fork** 버튼 클릭
3. Fork 완료

### 옵션 B: 로컬 Git 저장소 사용 (테스트용)

```bash
# 현재 디렉토리를 Git 저장소로 만들기
cd /home/ec2-user/helm/argo-helm

# 이미 Git 저장소이면 커밋
git add charts/argo-rollouts/examples/argocd-bluegreen-demo/
git commit -m "Add ArgoCD Blue-Green demo"

# Fork한 저장소나 자신의 저장소로 push
git remote set-url origin https://github.com/YOUR_USERNAME/argo-helm.git
git push origin main
```

## 4단계: ArgoCD Application 생성

### 방법 1: kubectl로 생성 (빠름)

먼저 `argocd-app.yaml` 파일을 수정합니다:

```yaml
# argocd-app.yaml에서 수정
source:
  repoURL: https://github.com/YOUR_USERNAME/argo-helm.git  # 자신의 저장소로 변경
  targetRevision: main
  path: charts/argo-rollouts/examples/argocd-bluegreen-demo/manifests
```

생성:

```bash
kubectl apply -f argocd-app.yaml
```

### 방법 2: ArgoCD UI로 생성 (쉬움)

1. ArgoCD 웹 UI 접속 (https://localhost:8080)
2. **+ NEW APP** 클릭
3. 정보 입력:
   - **Application Name**: `rollouts-bluegreen-demo`
   - **Project**: `default`
   - **Sync Policy**: `Automatic`
   - **Repository URL**: `https://github.com/YOUR_USERNAME/argo-helm.git`
   - **Revision**: `main`
   - **Path**: `charts/argo-rollouts/examples/argocd-bluegreen-demo/manifests`
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `rollouts-demo`
4. **CREATE** 클릭

### 방법 3: argocd CLI 사용

```bash
# argocd CLI 설치
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# ArgoCD 로그인
argocd login localhost:8080

# Application 생성
argocd app create rollouts-bluegreen-demo \
  --repo https://github.com/YOUR_USERNAME/argo-helm.git \
  --path charts/argo-rollouts/examples/argocd-bluegreen-demo/manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace rollouts-demo \
  --sync-policy automated
```

## 5단계: 배포 확인

### ArgoCD에서 확인

ArgoCD UI에서:
- Application 상태가 **Healthy** & **Synced** 확인
- 모든 리소스가 초록색으로 표시

### kubectl로 확인

```bash
# 네임스페이스 확인
kubectl get ns rollouts-demo

# 모든 리소스 확인
kubectl get all -n rollouts-demo

# Rollout 상태 확인
kubectl argo rollouts get rollout rollouts-demo -n rollouts-demo
```

## 6단계: 애플리케이션 접속 (Blue 버전)

```bash
# Active Service Port Forward
kubectl port-forward svc/rollouts-demo-active -n rollouts-demo 8888:80
```

브라우저에서 접속:
- **http://localhost:8888**
- 파란색 화면에 "BLUE Version" 표시

## 7단계: Green 버전 배포 (Blue → Green 전환)

### Green 버전으로 업데이트

rollout.yaml을 수정하여 Green ConfigMap을 사용하도록 변경:

```bash
# manifests/rollout.yaml 파일을 직접 수정하거나
# kubectl patch 사용

kubectl patch rollout rollouts-demo -n rollouts-demo --type merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "labels": {
          "version": "green"
        }
      },
      "spec": {
        "volumes": [{
          "name": "html",
          "configMap": {
            "name": "rollouts-demo-green"
          }
        }]
      }
    }
  }
}'
```

### Preview Service 확인

```bash
# 새 터미널에서 Preview Service Port Forward
kubectl port-forward svc/rollouts-demo-preview -n rollouts-demo 8889:80
```

브라우저에서 접속:
- **http://localhost:8889** (Preview - 테스트용)
- 초록색 화면에 "GREEN Version" 표시

이 시점에서:
- **Active (8888)**: Blue (운영 중) 👈 사용자 접속
- **Preview (8889)**: Green (테스트 중) 👈 검증 필요

## 8단계: 프로덕션 전환 (Promote)

Green 버전 테스트 완료 후:

```bash
# Promote 실행
kubectl argo rollouts promote rollouts-demo -n rollouts-demo
```

**확인**:
- http://localhost:8888 새로고침
- 파란색 → 초록색으로 변경됨!
- Active Service가 Green을 가리킴

30초 후 Blue Pod가 자동으로 삭제됩니다.

## 9단계: ArgoCD에서 모니터링

### ArgoCD UI
- Application 클릭
- 리소스 트리에서 Rollout 상태 확인
- Sync 이력 확인

### Argo Rollouts Dashboard

```bash
# Rollouts Dashboard 실행
kubectl argo rollouts dashboard

# 브라우저: http://localhost:3100
```

## 전체 워크플로우 요약

```
1. Git Push
   ↓
2. ArgoCD가 변경 감지
   ↓
3. Kubernetes에 자동 배포
   ↓
4. Rollout으로 Blue 배포
   ↓
5. Git에서 Green으로 변경
   ↓
6. ArgoCD가 자동 Sync
   ↓
7. Preview에 Green 배포
   ↓
8. 수동 테스트
   ↓
9. Promote 실행
   ↓
10. Active → Green 전환 완료!
```

## Git 기반 업데이트 방법

### 방법 1: Git에서 직접 수정 (GitOps 방식)

1. `manifests/rollout.yaml` 파일 수정:
   ```yaml
   volumes:
   - name: html
     configMap:
       name: rollouts-demo-green  # blue → green 변경
   ```

2. Git commit & push:
   ```bash
   git add manifests/rollout.yaml
   git commit -m "Update to Green version"
   git push origin main
   ```

3. ArgoCD가 자동으로 감지하고 배포 (Sync Policy가 Automatic인 경우)

### 방법 2: ArgoCD UI에서 Sync

1. ArgoCD UI 접속
2. Application 클릭
3. **SYNC** 버튼 클릭
4. 변경사항 확인 후 **SYNCHRONIZE** 클릭

## 트러블슈팅

### Application이 OutOfSync 상태

```bash
# 수동 Sync
argocd app sync rollouts-bluegreen-demo

# 또는 UI에서 SYNC 버튼 클릭
```

### Rollout이 Degraded 상태

```bash
# Rollout 상태 확인
kubectl describe rollout rollouts-demo -n rollouts-demo

# Rollout 재시작
kubectl argo rollouts restart rollouts-demo -n rollouts-demo
```

### Pod가 ImagePullBackOff

nginx:1.25-alpine 이미지는 공개 이미지이므로 문제없어야 합니다.
```bash
# Pod 로그 확인
kubectl logs -n rollouts-demo -l app=rollouts-demo

# 이미지 확인
kubectl describe pod -n rollouts-demo -l app=rollouts-demo
```

### ArgoCD에서 저장소 접근 불가

Private 저장소인 경우:
1. ArgoCD UI → Settings → Repositories
2. **CONNECT REPO** 클릭
3. GitHub 인증 정보 입력

## 고급 설정

### 자동 Promote 활성화

manifests/rollout.yaml 수정:

```yaml
strategy:
  blueGreen:
    autoPromotionEnabled: true
    autoPromotionSeconds: 60  # 60초 후 자동 전환
```

### Slack 알림 설정

ArgoCD에 Notification 설정:

```bash
# ArgoCD Notifications 설치
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/stable/manifests/install.yaml
```

## 정리

```bash
# Application 삭제
kubectl delete -f argocd-app.yaml

# 또는 ArgoCD CLI
argocd app delete rollouts-bluegreen-demo

# Namespace 삭제
kubectl delete namespace rollouts-demo
```

## 프로젝트 구조

```
argocd-bluegreen-demo/
├── manifests/                    # Kubernetes 리소스
│   ├── namespace.yaml           # rollouts-demo 네임스페이스
│   ├── configmap-blue.yaml      # Blue 버전 HTML
│   ├── configmap-green.yaml     # Green 버전 HTML
│   ├── services.yaml            # Active & Preview Services
│   └── rollout.yaml             # Argo Rollout 리소스
├── argocd-app.yaml              # ArgoCD Application 정의
└── README.md                    # 이 문서
```

## 학습 포인트

### GitOps
- Git이 단일 진실 공급원(Single Source of Truth)
- 코드 리뷰를 통한 배포 승인
- 변경 이력 추적

### Blue-Green 배포
- Preview에서 충분히 테스트
- 수동 승인 후 전환
- 빠른 롤백 가능

### ArgoCD + Argo Rollouts
- ArgoCD: 배포 자동화
- Argo Rollouts: 배포 전략
- 완벽한 GitOps 구현

## 다음 단계

1. **Canary 배포**: 점진적 트래픽 전환
2. **Analysis**: Prometheus 메트릭 기반 자동 승인
3. **Multi-cluster**: 여러 클러스터에 배포
4. **CI/CD 통합**: GitHub Actions와 통합

## 참고 자료

- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/)
- [Argo Rollouts 공식 문서](https://argoproj.github.io/argo-rollouts/)
- [Blue-Green 배포 가이드](https://argoproj.github.io/argo-rollouts/features/bluegreen/)

## 문의

이슈나 질문이 있으시면 GitHub Issues에 남겨주세요!
