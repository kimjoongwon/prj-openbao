# OpenBao CLI 빠른 시작 가이드

## 🚀 3단계로 시작하기 (5분)

### 1단계: CLI 설치 및 연결

```bash
# CLI 자동 설치 (macOS/Linux)
./scripts/install-vault-cli.sh

# OpenBao 서버 연결
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 서버 상태 확인
vault status

# 로그인
vault login
```

### 2단계: 시크릿 조회

```bash
# 서버 환경 시크릿 목록
vault kv list secret/server

# Staging 환경 시크릿 읽기
vault kv get secret/server/staging

# Production 환경 시크릿 읽기
vault kv get secret/server/production

# Harbor 시크릿 조회
vault kv get secret/harbor/staging
vault kv get secret/harbor/production
```

### 3단계: 시크릿 생성 (관리자용)

```bash
# 헬퍼 스크립트 사용 (대화형)
./scripts/create-secrets.sh

# 또는 수동 생성
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  NODE_ENV=staging \
  DATABASE_URL=postgresql://user:pass@host:5432/db
```

---

## 📋 자주 사용하는 명령어

### 시크릿 조회
```bash
# 목록 보기
vault kv list secret/server
vault kv list secret/harbor

# 특정 시크릿 읽기
vault kv get secret/server/staging
vault kv get secret/harbor/production

# JSON 형식으로 출력
vault kv get -format=json secret/server/staging

# 특정 필드만 추출
vault kv get -field=APP_PORT secret/server/staging
```

### 시크릿 생성/수정
```bash
# 새 시크릿 생성
vault kv put secret/server/default \
  APP_PORT=3000 \
  APP_NAME=my-app

# 기존 시크릿에 키 추가 (merge)
vault kv patch secret/server/staging \
  NEW_KEY=new_value

# 전체 덮어쓰기
vault kv put secret/server/staging \
  APP_PORT=3001 \
  APP_NAME=updated-app
```

### 시크릿 삭제
```bash
# 최신 버전 삭제 (soft delete, 복구 가능)
vault kv delete secret/server/staging

# 완전 삭제 (복구 불가)
vault kv destroy -versions=1 secret/server/staging

# 메타데이터 포함 완전 삭제
vault kv metadata delete secret/server/staging
```

### 버전 관리
```bash
# 변경 이력 보기
vault kv metadata get secret/server/staging

# 특정 버전 읽기
vault kv get -version=2 secret/server/staging

# 이전 버전으로 롤백
vault kv rollback -version=1 secret/server/staging
```

---

## 🔐 정책 및 토큰 관리 (관리자용)

### 정책 생성
```bash
# 자동 설정 스크립트 사용 (권장)
cd /Users/wallykim/dev/prj-openbao
chmod +x scripts/setup-esc.sh
./scripts/setup-esc.sh

# 수동 설정
vault policy write esc-policy policies/esc-policy.hcl
vault policy read esc-policy
```

### 토큰 관리
```bash
# 현재 토큰 정보 확인
vault token lookup

# 다른 토큰 정보 확인
vault token lookup <token>

# 토큰 갱신
vault token renew

# 토큰 폐기
vault token revoke <token>

# 새 토큰 생성
vault token create \
  -policy=esc-policy \
  -ttl=720h \
  -period=24h \
  -display-name=team-member-token
```

---

## 🔨 시크릿 생성 예제

### Harbor Docker Registry 인증

```bash
# Staging Harbor 시크릿
vault kv put secret/harbor/staging \
  .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{"username":"robot$staging","password":"your-token","auth":"base64-encoded"}}}'

# Production Harbor 시크릿
vault kv put secret/harbor/production \
  .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{"username":"robot$production","password":"your-token","auth":"base64-encoded"}}}'
```

### 서버 환경 변수

```bash
# Staging 환경
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  APP_ADMIN_EMAIL=admin@example.com \
  API_PREFIX=/api \
  APP_FALLBACK_LANGUAGE=ko \
  FRONTEND_DOMAIN=https://staging.example.com \
  BACKEND_DOMAIN=https://api-staging.example.com \
  NODE_ENV=staging \
  DATABASE_URL=postgresql://user:pass@staging-db:5432/db \
  AWS_ACCESS_KEY_ID=your-staging-key \
  AWS_SECRET_ACCESS_KEY=your-staging-secret \
  AWS_REGION=ap-northeast-2 \
  SMTP_HOST=smtp.example.com \
  SMTP_PORT=587 \
  AUTH_JWT_SECRET=staging-jwt-secret

# Production 환경
vault kv put secret/server/production \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  NODE_ENV=production \
  DATABASE_URL=postgresql://user:pass@prod-db:5432/db \
  # ... production 값들
```

---

## 🔄 시크릿 업데이트 워크플로우

### 전체 업데이트
```bash
# 1. 현재 값 확인
vault kv get secret/server/staging

# 2. 새 값으로 덮어쓰기
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=new-value \
  # ... 모든 키-값 쌍

# 3. 업데이트 확인
vault kv get secret/server/staging
```

### 부분 업데이트
```bash
# 특정 키만 업데이트 (다른 키는 유지)
vault kv patch secret/server/staging \
  APP_NAME=updated-value \
  DATABASE_URL=new-db-url

# 확인
vault kv get secret/server/staging
```

### 실수한 경우 롤백
```bash
# 변경 이력 확인
vault kv metadata get secret/server/staging

# 이전 버전으로 복구
vault kv rollback -version=1 secret/server/staging
```

---

## 🚨 문제 해결

### "permission denied" 오류
```bash
# 1. 현재 토큰의 정책 확인
vault token lookup | grep policies

# 2. 정책 내용 확인
vault policy read esc-policy

# 3. 필요시 정책 재적용
vault policy write esc-policy policies/esc-policy.hcl
```

### 연결 오류
```bash
# 1. 환경 변수 확인
echo $VAULT_ADDR

# 2. 서버 상태 확인
vault status

# 3. 네트워크 연결 테스트
curl $VAULT_ADDR/v1/sys/health
```

### 토큰 만료
```bash
# 1. 토큰 TTL 확인
vault token lookup | grep ttl

# 2. 토큰 갱신
vault token renew

# 3. 새 토큰 생성 (만료된 경우)
vault login
```

### 시크릿이 없음
```bash
# 1. 경로 확인
vault kv list secret/server

# 2. 메타데이터 확인
vault kv metadata get secret/server/staging

# 3. 시크릿 생성
./scripts/create-secrets.sh staging
```

### 삭제한 시크릿 복구
```bash
# 1. 삭제 이력 확인
vault kv metadata get secret/server/staging

# 2. 삭제 취소 (soft delete된 경우)
vault kv undelete -versions=1 secret/server/staging

# 3. 이전 버전으로 롤백
vault kv rollback -version=1 secret/server/staging
```

---

## 📚 추가 문서

- **상세 가이드**: [README.md](README.md)
- **CLI 설치**: [INSTALL-CLI.md](INSTALL-CLI.md)
- **외부 접근**: [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md)
- **전체 문서**: [INDEX.md](INDEX.md)
- **Kubernetes 통합** (선택): [KUBERNETES.md](KUBERNETES.md)

## 🔐 보안 체크리스트

- [ ] 토큰이 Git에 커밋되지 않음 (.gitignore 확인)
- [ ] Production과 Staging 토큰 분리 (권장)
- [ ] 정기적인 토큰 교체 일정 수립 (3-6개월)
- [ ] 최소 권한 원칙 적용 (필요한 경로만 접근)
- [ ] 민감한 시크릿은 안전한 채널로만 공유

## 💡 팁

### 환경 변수 설정 저장
```bash
# ~/.zshrc 또는 ~/.bashrc에 추가
export VAULT_ADDR=https://openbao.cocdev.co.kr
export VAULT_TOKEN=your-token  # 주의: 개발 환경에서만 사용
```

### JSON 출력 활용
```bash
# jq와 함께 사용
vault kv get -format=json secret/server/staging | jq '.data.data'

# 특정 필드 추출
vault kv get -format=json secret/server/staging | jq -r '.data.data.APP_PORT'
```

### 자동화 스크립트
```bash
# 여러 환경의 같은 키 비교
for env in staging production; do
  echo "$env APP_PORT:"
  vault kv get -field=APP_PORT secret/server/$env
done
```
