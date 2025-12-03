# OpenBao Kubernetes 통합 가이드

> ⚠️ **주의**: 이 문서는 선택적 고급 기능입니다.
> 기본적인 CLI 사용만 필요한 경우 [README.md](README.md)를 참고하세요.

---

## 개요

Kubernetes 환경에서 External Secrets Operator(ESC)를 사용하여 OpenBao의 시크릿을 자동으로 동기화하는 방법을 설명합니다.

**전제 조건**:
- Kubernetes 클러스터 접근 권한
- `kubectl` 설치 및 설정
- External Secrets Operator 설치
- OpenBao 관리자 권한

---

## ESC (External Secrets Controller) 설정

### 1단계: 정책 및 토큰 생성

```bash
# OpenBao에 로그인 (관리자)
export VAULT_ADDR=https://openbao.cocdev.co.kr
vault login

# 자동 설정 스크립트 실행
cd /Users/wallykim/dev/prj-openbao
chmod +x scripts/setup-esc.sh
./scripts/setup-esc.sh

# 출력된 토큰 저장
```

**수동 설정**:
```bash
# 정책 적용
vault policy write esc-policy policies/esc-policy.hcl

# 토큰 생성 (30일, 자동 갱신)
vault token create \
  -policy=esc-policy \
  -ttl=720h \
  -period=24h \
  -renewable=true \
  -display-name=esc-token
```

### 2단계: OpenBao에 시크릿 생성

```bash
# 헬퍼 스크립트 사용
./scripts/create-secrets.sh staging
./scripts/create-secrets.sh production

# 또는 수동 생성
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  NODE_ENV=staging \
  DATABASE_URL=postgresql://...

vault kv put secret/harbor/staging \
  .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{...}}}'
```

### 3단계: Kubernetes Secret 생성

```bash
# 1단계에서 받은 토큰 사용
OPENBAO_TOKEN="<생성된_토큰>"

# Staging 환경
kubectl create secret generic openbao-token \
  --from-literal=token=$OPENBAO_TOKEN \
  --namespace=external-secrets-stg \
  --dry-run=client -o yaml | kubectl apply -f -

# Production 환경
kubectl create secret generic openbao-token \
  --from-literal=token=$OPENBAO_TOKEN \
  --namespace=external-secrets-prod \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## ClusterSecretStore 설정

### ClusterSecretStore 생성

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: openbao-backend
spec:
  provider:
    vault:
      server: "http://openbao.openbao-system.svc.cluster.local:8200"
      # 또는 외부 접근: "https://openbao.cocdev.co.kr"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: "openbao-token"
          namespace: "external-secrets-stg"
          key: "token"
```

적용:
```bash
kubectl apply -f secretstore.yaml
kubectl get clustersecretstore
```

---

## ExternalSecret 설정

### 서버 환경 변수 ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-env-secrets-staging
  namespace: plate-api-stg
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: openbao-backend
    kind: ClusterSecretStore
  target:
    name: app-env-secrets
    creationPolicy: Owner
  data:
  - secretKey: APP_PORT
    remoteRef:
      key: secret/server/staging
      property: APP_PORT
  - secretKey: APP_NAME
    remoteRef:
      key: secret/server/staging
      property: APP_NAME
  - secretKey: DATABASE_URL
    remoteRef:
      key: secret/server/staging
      property: DATABASE_URL
```

### Harbor Docker Secret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-docker-secret-staging
  namespace: plate-api-stg
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: openbao-backend
    kind: ClusterSecretStore
  target:
    name: harbor-docker-secret
    creationPolicy: Owner
    template:
      type: kubernetes.io/dockerconfigjson
      data:
        .dockerconfigjson: "{{ .dockerconfigjson }}"
  data:
  - secretKey: dockerconfigjson
    remoteRef:
      key: secret/harbor/staging
      property: .dockerconfigjson
```

적용:
```bash
kubectl apply -f externalsecret-server.yaml
kubectl apply -f externalsecret-harbor.yaml

# 상태 확인
kubectl get externalsecret -n plate-api-stg
kubectl describe externalsecret app-env-secrets-staging -n plate-api-stg
```

---

## 동작 확인

### SecretStore 확인
```bash
kubectl get secretstore -n plate-api-stg
kubectl get clustersecretstore

# 상세 정보
kubectl describe secretstore openbao-backend
```

### ExternalSecret 확인
```bash
kubectl get externalsecret -n plate-api-stg
kubectl describe externalsecret app-env-secrets-staging -n plate-api-stg
```

### 생성된 Secret 확인
```bash
# Secret 존재 확인
kubectl get secret app-env-secrets -n plate-api-stg
kubectl get secret harbor-docker-secret -n plate-api-stg

# Secret 내용 확인 (base64 디코딩)
kubectl get secret app-env-secrets -n plate-api-stg -o jsonpath='{.data.APP_NAME}' | base64 -d
echo

# 전체 Secret 보기
kubectl get secret app-env-secrets -n plate-api-stg -o yaml
```

---

## 시크릿 업데이트 워크플로우

### 1. OpenBao 시크릿 수정

```bash
# 전체 업데이트
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=new-value \
  DATABASE_URL=new-url

# 부분 업데이트 (patch)
vault kv patch secret/server/staging \
  APP_NAME=new-value \
  DATABASE_URL=new-db-url
```

### 2. 강제 동기화 (대기 시간 없이)

**방법 1: ExternalSecret 삭제 후 재생성**
```bash
kubectl delete externalsecret app-env-secrets-staging -n plate-api-stg
# ESC가 자동으로 재생성함
```

**방법 2: Annotation 추가로 강제 갱신**
```bash
kubectl annotate externalsecret app-env-secrets-staging \
  force-sync="$(date +%s)" \
  -n plate-api-stg
```

### 3. Pod 재시작 (필요시)

```bash
# Deployment 재시작
kubectl rollout restart deployment/<deployment-name> -n plate-api-stg

# 상태 확인
kubectl rollout status deployment/<deployment-name> -n plate-api-stg
```

---

## Helm 차트 배포 (예제)

> **주의**: 실제 Helm 차트 파일이 필요합니다.

### Staging 배포
```bash
helm upgrade --install openbao-secrets-manager \
  ./helm/shared-configs/openbao-secrets-manager \
  -f ./helm/shared-configs/openbao-secrets-manager/values-staging.yaml \
  --namespace plate-stg \
  --create-namespace
```

### Production 배포
```bash
helm upgrade --install openbao-secrets-manager \
  ./helm/shared-configs/openbao-secrets-manager \
  -f ./helm/shared-configs/openbao-secrets-manager/values-production.yaml \
  --namespace plate-prod \
  --create-namespace
```

---

## 트러블슈팅

### "permission denied" 오류

```bash
# 1. 토큰 정책 확인
vault token lookup <esc-token> | grep policies

# 2. 정책 재적용
vault policy write esc-policy policies/esc-policy.hcl

# 3. Kubernetes Secret 확인
kubectl get secret openbao-token -n external-secrets-stg -o jsonpath='{.data.token}' | base64 -d
```

### ExternalSecret이 동기화되지 않음

```bash
# 1. External Secrets Operator 로그 확인
kubectl logs -n external-secrets deployment/external-secrets -f

# 2. SecretStore 연결 테스트
kubectl get secretstore -n plate-api-stg -o yaml

# 3. ExternalSecret 상태 확인
kubectl describe externalsecret app-env-secrets-staging -n plate-api-stg

# 4. OpenBao 토큰 확인
kubectl get secret openbao-token -n external-secrets-stg -o jsonpath='{.data.token}' | base64 -d | \
  xargs -I {} vault token lookup {}
```

### Secret이 생성되지 않음

```bash
# 1. OpenBao에 시크릿이 존재하는지 확인
vault kv list secret/server/
vault kv get secret/server/staging

# 2. 없다면 생성
./scripts/create-secrets.sh staging

# 3. ExternalSecret 이벤트 확인
kubectl get events -n plate-api-stg --field-selector involvedObject.name=app-env-secrets-staging
```

### 토큰 만료

```bash
# 1. 토큰 TTL 확인
vault token lookup <esc-token> | grep ttl

# 2. 토큰 갱신 (period 설정된 경우 자동 갱신)
vault token renew <esc-token>

# 3. 새 토큰 생성
vault token create -policy=esc-policy -ttl=720h -period=24h

# 4. Kubernetes Secret 업데이트
kubectl create secret generic openbao-token \
  --from-literal=token=<new-token> \
  --namespace=external-secrets-stg \
  --dry-run=client -o yaml | kubectl apply -f -

# 5. ESC Pod 재시작
kubectl rollout restart deployment external-secrets -n external-secrets
```

---

## 정책 커스터마이징

`policies/esc-policy.hcl` 파일을 수정하여 접근 경로를 추가/제거:

```hcl
# 새로운 애플리케이션별 경로 추가
path "secret/data/plate-api/*" {
  capabilities = ["read", "list"]
}

path "secret/metadata/plate-api/*" {
  capabilities = ["read", "list"]
}

# 다른 애플리케이션
path "secret/data/plate-web/*" {
  capabilities = ["read", "list"]
}
```

수정 후 정책 재적용:
```bash
vault policy write esc-policy policies/esc-policy.hcl
```

---

## 보안 권장사항

### 1. 최소 권한 원칙
- ESC 토큰은 읽기 전용 권한만 보유
- 필요한 경로만 정책에 포함

### 2. 토큰 주기 관리
- `period` 설정으로 자동 갱신 활성화
- 정기적인 토큰 교체 (3-6개월)

### 3. 토큰 저장
- Kubernetes Secret에만 저장
- Git에 커밋하지 않음 (.gitignore 확인)

### 4. 환경 분리
- Production과 Staging 토큰 분리
- Namespace 분리

### 5. 감사 로그
```bash
vault audit enable file file_path=/var/log/vault/audit.log
```

---

## 참고 자료

- [External Secrets Operator Documentation](https://external-secrets.io/latest/)
- [External Secrets - Vault Provider](https://external-secrets.io/latest/provider/hashicorp-vault/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Helm](https://helm.sh/docs/)

---

## 추가 문서

- **메인 가이드**: [README.md](README.md)
- **빠른 시작**: [QUICKSTART.md](QUICKSTART.md)
- **CLI 설치**: [INSTALL-CLI.md](INSTALL-CLI.md)
- **외부 접근**: [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md)
- **전체 인덱스**: [INDEX.md](INDEX.md)
