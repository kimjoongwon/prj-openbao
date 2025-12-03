# OpenBao ESC 빠른 시작 가이드

## 🚀 3단계로 ESC 설정하기

### 1단계: 정책 및 토큰 생성

```bash
# OpenBao 로그인
export VAULT_ADDR=https://openbao.cocdev.co.kr
vault login

# 자동 설정 스크립트 실행
cd /Users/wallykim/dev/prj-openbao
chmod +x scripts/setup-esc.sh
./scripts/setup-esc.sh
```

**출력된 토큰을 저장하세요!**

### 2단계: OpenBao에 시크릿 생성

```bash
# Staging 시크릿 생성
chmod +x scripts/create-secrets.sh
./scripts/create-secrets.sh staging

# Production 시크릿 생성
./scripts/create-secrets.sh production
```

### 3단계: Kubernetes Secret 생성

```bash
# 1단계에서 받은 토큰 사용
OPENBAO_TOKEN="<1단계에서_생성된_토큰>"

# Staging 환경
kubectl create secret generic openbao-token \
  --from-literal=token=$OPENBAO_TOKEN \
  --namespace=plate-stg \
  --dry-run=client -o yaml | kubectl apply -f -

# Production 환경
kubectl create secret generic openbao-token \
  --from-literal=token=$OPENBAO_TOKEN \
  --namespace=plate-prod \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 📋 현재 설정 확인

### OpenBao 정책 확인
```bash
vault policy read esc-policy
```

### 토큰 정보 확인
```bash
vault token lookup <토큰>
```

### 시크릿 확인
```bash
# 서버 시크릿
vault kv get secret/server/staging
vault kv get secret/server/production

# Harbor 시크릿
vault kv get secret/harbor/staging
vault kv get secret/harbor/production
```

## 🔧 Helm 차트 배포

### Staging
```bash
helm upgrade --install openbao-secrets-manager \
  ./helm/shared-configs/openbao-secrets-manager \
  -f ./helm/shared-configs/openbao-secrets-manager/values-staging.yaml \
  --namespace plate-stg \
  --create-namespace
```

### Production
```bash
helm upgrade --install openbao-secrets-manager \
  ./helm/shared-configs/openbao-secrets-manager \
  -f ./helm/shared-configs/openbao-secrets-manager/values-production.yaml \
  --namespace plate-prod \
  --create-namespace
```

## 🧪 동작 확인

### SecretStore 확인
```bash
kubectl get secretstore -n plate-stg
kubectl get secretstore -n plate-prod
```

### ExternalSecret 확인
```bash
kubectl get externalsecret -n plate-stg
kubectl get externalsecret -n plate-prod
```

### 생성된 Secret 확인
```bash
kubectl get secret app-env-secrets -n plate-stg
kubectl get secret harbor-docker-secret -n plate-stg

kubectl get secret app-env-secrets -n plate-prod
kubectl get secret harbor-docker-secret -n plate-prod
```

### 상세 확인 (디버깅)
```bash
# ExternalSecret 상태 확인
kubectl describe externalsecret app-env-secrets-staging -n plate-stg

# SecretStore 상태 확인
kubectl describe secretstore openbao-env-staging -n plate-stg

# Secret 내용 확인 (base64 디코딩)
kubectl get secret app-env-secrets -n plate-stg -o jsonpath='{.data.APP_NAME}' | base64 -d
```

## 🔄 시크릿 업데이트

### OpenBao 시크릿 수정
```bash
# 전체 업데이트
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=new-value \
  ...

# 부분 업데이트 (patch)
vault kv patch secret/server/staging \
  APP_NAME=new-value \
  DATABASE_URL=new-db-url
```

### 강제 동기화 (대기 시간 없이)
```bash
# ExternalSecret 삭제 후 재생성 (자동으로 다시 생성됨)
kubectl delete externalsecret app-env-secrets-staging -n plate-stg

# 또는 annotation 추가로 강제 갱신
kubectl annotate externalsecret app-env-secrets-staging \
  force-sync="$(date +%s)" \
  -n plate-stg
```

## 🚨 문제 해결

### "permission denied" 오류
```bash
# 토큰 정책 확인
vault token lookup | grep policies

# 정책 재적용
vault policy write esc-policy openbao/policies/esc-policy.hcl
```

### ExternalSecret이 동기화되지 않음
```bash
# 로그 확인
kubectl logs -n external-secrets deployment/external-secrets -f

# SecretStore 연결 테스트
kubectl get secretstore -n plate-stg -o yaml

# OpenBao 토큰 확인
kubectl get secret openbao-token -n plate-stg -o jsonpath='{.data.token}' | base64 -d
```

### 시크릿이 없음
```bash
# OpenBao에 시크릿이 존재하는지 확인
vault kv list secret/server/
vault kv get secret/server/staging

# 없다면 생성
./scripts/create-secrets.sh staging
```

## 📚 추가 정보

- 상세 가이드: [README.md](README.md)
- 정책 파일: [policies/esc-policy.hcl](policies/esc-policy.hcl)
- 설정 스크립트: [scripts/setup-esc.sh](scripts/setup-esc.sh)
- 시크릿 생성: [scripts/create-secrets.sh](scripts/create-secrets.sh)

## 🔐 보안 체크리스트

- [ ] ESC 토큰은 읽기 전용 권한만 보유
- [ ] 토큰이 Git에 커밋되지 않음
- [ ] Production과 Staging 토큰이 분리됨 (권장)
- [ ] 정기적인 토큰 교체 일정 수립 (3-6개월)
- [ ] OpenBao 감사 로그 활성화
- [ ] Kubernetes Secret은 암호화됨 (etcd encryption)

## 📞 지원

문제가 발생하면 다음을 확인하세요:
1. OpenBao 로그
2. External Secrets Operator 로그
3. Kubernetes 이벤트
4. 이 가이드의 문제 해결 섹션
