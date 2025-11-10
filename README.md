# Argo Rollouts Blue-Green Deployment Demo

ArgoCD + Argo Rollouts를 사용한 Blue-Green 배포 데모입니다.

## 특징

- ✅ **빌드 불필요**: 공개 nginx 이미지 사용 (ECR, Docker 빌드 불필요)
- ✅ **ArgoCD GitOps**: Git 저장소 기반 자동 배포
- ✅ **Blue-Green 전략**: 안전한 무중단 배포
- ✅ **시각적 확인**: Blue(파란색) ↔ Green(초록색)
- ✅ **간단한 구조**: Kubernetes 초보자도 이해 가능

## 프로젝트 구조

```
argo-rollouts-demo/
└── argocd-bluegreen-demo/
    ├── manifests/               # 모든 Kubernetes 리소스
    │   ├── namespace.yaml      # rollouts-demo 네임스페이스
    │   ├── configmap-blue.yaml # Blue 버전 (파란색)
    │   ├── configmap-green.yaml# Green 버전 (초록색)
    │   ├── services.yaml       # Active & Preview Services
    │   └── rollout.yaml        # Argo Rollout 설정
    ├── argocd-app.yaml         # ArgoCD Application 정의
    └── README.md               # 상세 가이드
```

## 빠른 시작

### 1. 사전 요구사항

- Kubernetes 클러스터
- kubectl 설치
- Argo Rollouts 설치
- ArgoCD 설치

### 2. Argo Rollouts 설치

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argo-rollouts argo/argo-rollouts -n argo-rollouts --create-namespace
```

### 3. ArgoCD 설치

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. ArgoCD 접속

```bash
# Port Forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 초기 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

브라우저에서 https://localhost:8080 접속
- Username: `admin`
- Password: (위에서 확인한 비밀번호)

### 5. Application 배포

#### 방법 1: kubectl 사용

```bash
# argocd-app.yaml 파일의 repoURL을 자신의 저장소로 수정 후
kubectl apply -f argocd-bluegreen-demo/argocd-app.yaml
```

#### 방법 2: ArgoCD UI 사용

1. ArgoCD UI에서 **+ NEW APP** 클릭
2. 정보 입력:
   - **Application Name**: `rollouts-bluegreen-demo`
   - **Project**: `default`
   - **Repository URL**: `https://github.com/YOUR_USERNAME/argo-rollouts-demo.git`
   - **Path**: `argocd-bluegreen-demo/manifests`
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `rollouts-demo`
3. **CREATE** 클릭

### 6. Blue 버전 접속

```bash
kubectl port-forward svc/rollouts-demo-active -n rollouts-demo 8888:80
```

브라우저: http://localhost:8888 → 파란색 화면

### 7. Green 버전 배포

```bash
kubectl patch rollout rollouts-demo -n rollouts-demo --type merge -p '{
  "spec": {
    "template": {
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

### 8. Preview 확인

```bash
kubectl port-forward svc/rollouts-demo-preview -n rollouts-demo 8889:80
```

브라우저: http://localhost:8889 → 초록색 화면

### 9. 프로덕션 전환

```bash
kubectl argo rollouts promote rollouts-demo -n rollouts-demo
```

브라우저: http://localhost:8888 새로고침 → 초록색으로 변경!

## Blue-Green 배포 워크플로우

```
[1. Blue 배포]
Active (8888)  → Blue (파란색)   ← 사용자 접속
Preview        → 없음

↓ Green 배포

[2. Green 테스트]
Active (8888)  → Blue (파란색)   ← 사용자 접속 (운영 유지)
Preview (8889) → Green (초록색)  ← 테스트

↓ Promote

[3. 전환 완료]
Active (8888)  → Green (초록색)  ← 전환됨!
Preview (8889) → Green (초록색)
Blue          → 30초 후 삭제
```

## 상세 가이드

전체 가이드는 [argocd-bluegreen-demo/README.md](./argocd-bluegreen-demo/README.md)를 참고하세요.

## 참고 자료

- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/)
- [Argo Rollouts 공식 문서](https://argoproj.github.io/argo-rollouts/)
- [Blue-Green 배포 가이드](https://argoproj.github.io/argo-rollouts/features/bluegreen/)

## 라이선스

MIT License
