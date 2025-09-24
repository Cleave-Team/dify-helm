# Dify Helm Chart ArgoCD 배포 가이드

Fork한 Dify Helm Chart를 ArgoCD로 배포하는 방법입니다.

## 사전 준비

1. Fork한 저장소 준비
   ```bash
   git clone https://github.com/YOUR_USERNAME/dify-helm.git
   cd dify-helm
   ```

2. 변경사항 커밋 및 푸시
   ```bash
   git add .
   git commit -m "feat: Add keepalive_timeout configuration"
   git push origin master
   ```

## ArgoCD Application 배포

### 1. 기본 배포

```bash
# ArgoCD Application 생성
kubectl apply -f argocd-application.yaml
```

### 2. Secret을 포함한 배포

```bash
# Secret 먼저 생성
kubectl create secret generic dify-secrets \
  --namespace=argocd \
  --from-literal=dify.secretKey="$(openssl rand -base64 42)" \
  --from-literal=dify.dbPassword="your-db-password" \
  --from-literal=dify.redisPassword="your-redis-password" \
  --from-literal=dify.s3AccessKey="your-s3-access-key" \
  --from-literal=dify.s3SecretKey="your-s3-secret-key"

# Application 배포
kubectl apply -f argocd-application-with-secrets.yaml
```

### 3. ArgoCD CLI를 사용한 배포

```bash
# ArgoCD 로그인
argocd login argocd.example.com

# Application 생성
argocd app create dify \
  --repo https://github.com/YOUR_USERNAME/dify-helm \
  --path charts/dify \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dify \
  --revision master \
  --helm-set proxy.keepaliveTimeout=120 \
  --helm-set proxy.replicas=2 \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Sync 실행
argocd app sync dify
```

## 환경별 배포

### Development 환경

```yaml
# argocd-app-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dify-dev
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/YOUR_USERNAME/dify-helm
    targetRevision: develop
    path: charts/dify
    helm:
      valueFiles:
        - values.yaml
        - values-dev.yaml
```

### Production 환경

```yaml
# argocd-app-prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dify-prod
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/YOUR_USERNAME/dify-helm
    targetRevision: master
    path: charts/dify
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
```

## 커스텀 설정 사용

### keepalive_timeout 변경

values 파일에서 설정:
```yaml
proxy:
  keepaliveTimeout: 180  # 초 단위
```

또는 ArgoCD Application에서 직접 설정:
```yaml
spec:
  source:
    helm:
      parameters:
        - name: proxy.keepaliveTimeout
          value: "180"
```

## 모니터링

### Application 상태 확인

```bash
# CLI
argocd app get dify
argocd app health dify

# 로그 확인
argocd app logs dify --follow

# 히스토리 확인
argocd app history dify
```

### ArgoCD UI에서 확인

1. ArgoCD UI 접속: https://argocd.example.com
2. Applications > dify 클릭
3. 배포 상태 및 리소스 확인

## 롤백

```bash
# 이전 버전 확인
argocd app history dify

# 특정 버전으로 롤백
argocd app rollback dify 2

# 또는 UI에서:
# History 탭 > 원하는 버전 선택 > Rollback
```

## 문제 해결

### Sync 실패 시

```bash
# 상태 확인
kubectl describe application dify -n argocd

# 강제 Sync
argocd app sync dify --force

# 리소스 정리 후 재배포
argocd app delete dify
kubectl apply -f argocd-application.yaml
```

### Secret 관련 문제

```bash
# Secret 확인
kubectl get secret dify-secrets -n argocd -o yaml

# Secret 업데이트
kubectl edit secret dify-secrets -n argocd
```

## 추가 리소스

- [ArgoCD 공식 문서](https://argo-cd.readthedocs.io/)
- [Dify Helm Chart 문서](https://github.com/douban/dify-helm)
- [Kubernetes Secrets 관리](https://kubernetes.io/docs/concepts/configuration/secret/)