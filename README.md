# OpenBao 관리 프로젝트

OpenBao CLI를 사용한 시크릿 관리 프로젝트입니다. 팀원들이 빠르게 랜딩하여 Harbor 시크릿과 Staging/Production 환경 시크릿을 관리할 수 있도록 돕습니다.

## 📋 프로젝트 구조

```
prj-openbao/
├── 📚 문서
│   ├── INDEX.md                           # 전체 문서 인덱스
│   ├── QUICKSTART.md                      # CLI 빠른 시작 (3단계)
│   ├── INSTALL-CLI.md                     # Vault CLI 빠른 설치
│   ├── EXTERNAL-ACCESS-QUICKSTART.md      # 외부 접근 가이드
│   ├── KUBERNETES.md                      # Kubernetes 통합 (선택)
│   └── docs/
│       ├── vault-cli-installation.md      # CLI 상세 설치
│       └── external-access.md             # 외부 접근 상세
│
├── 🔐 정책 및 스크립트
│   ├── policies/
│   │   └── esc-policy.hcl                # 읽기 전용 정책
│   └── scripts/
│       ├── install-vault-cli.sh          # CLI 자동 설치
│       ├── setup-esc.sh                  # 정책/토큰 자동 생성
│       └── create-secrets.sh             # 시크릿 생성 헬퍼
```

## 🚀 빠른 시작 (5분)

### 1. Vault CLI 설치
```bash
./scripts/install-vault-cli.sh
```

### 2. OpenBao 연결 및 로그인
```bash
export VAULT_ADDR=https://openbao.cocdev.co.kr
vault status
vault login
```

### 3. 시크릿 읽기
```bash
# Staging 환경 시크릿 조회
vault kv get secret/server/staging

# Production 환경 시크릿 조회
vault kv get secret/server/production

# Harbor 시크릿 조회
vault kv get secret/harbor/staging
```

### 4. 자세한 가이드
- **CLI 설치**: [INSTALL-CLI.md](INSTALL-CLI.md)
- **외부 접근**: [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md)
- **빠른 시작**: [QUICKSTART.md](QUICKSTART.md)
- **전체 문서**: [INDEX.md](INDEX.md)

---

## 🔑 주요 CLI 명령어

### 연결 및 인증

```bash
# 서버 상태 확인
vault status

# 로그인
vault login

# 토큰 정보 확인
vault token lookup
```

### 시크릿 조회

```bash
# 시크릿 목록 보기
vault kv list secret/server
vault kv list secret/harbor

# 특정 시크릿 읽기
vault kv get secret/server/staging/config
vault kv get secret/server/production/config
vault kv get secret/harbor/staging/auth
vault kv get secret/harbor/production/auth

# JSON 형식으로 출력
vault kv get -format=json secret/server/staging
```

### 시크릿 생성/수정

```bash
# 새 시크릿 생성
vault kv put secret/server/staging/config \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  NODE_ENV=staging

# 기존 시크릿에 키 추가 (merge)
vault kv patch secret/server/staging/config \
  NEW_KEY=new_value

# 전체 시크릿 덮어쓰기
vault kv put secret/server/staging/config \
  APP_PORT=3001 \
  APP_NAME=plate-api-v2
```

### 시크릿 삭제

```bash
# 최신 버전 삭제 (soft delete)
vault kv delete secret/server/staging/config

# 특정 버전 삭제
vault kv delete -versions=2,3 secret/server/staging/config

# 완전 삭제 (복구 불가)
vault kv destroy -versions=1,2 secret/server/staging/config

# 메타데이터 포함 완전 삭제
vault kv metadata delete secret/server/staging/config
```

### 버전 관리

```bash
# 시크릿 변경 이력 보기
vault kv metadata get secret/server/staging/config

# 특정 버전 읽기
vault kv get -version=2 secret/server/staging/config

# 이전 버전으로 복구
vault kv rollback -version=1 secret/server/staging/config
```

---

## 📁 관리되는 시크릿 경로

### 서버 환경 변수
- `secret/server/staging` - Staging 환경 설정
- `secret/server/production` - Production 환경 설정
- `secret/server/default` - 기본 환경 설정

### Harbor Registry 인증
- `secret/harbor/staging` - Staging Harbor 인증
- `secret/harbor/production` - Production Harbor 인증
- `secret/harbor/development` - Development Harbor 인증

---

## 🔐 정책 및 토큰 관리

### 정책 생성 (관리자용)

```bash
# 정책 파일 적용
vault policy write esc-policy policies/esc-policy.hcl

# 정책 확인
vault policy read esc-policy

# 정책 목록
vault policy list
```

### 토큰 생성 (관리자용)

```bash
# 자동 생성 스크립트 사용 (권장)
./scripts/setup-esc.sh

# 수동 생성 - 기본 토큰 (30일, 자동 갱신)
vault token create \
  -policy=esc-policy \
  -ttl=720h \
  -period=24h \
  -renewable=true \
  -display-name=team-token
```

### 토큰 관리

```bash
# 토큰 정보 확인
vault token lookup

# 다른 토큰 정보 확인
vault token lookup <token>

# 토큰 갱신
vault token renew

# 토큰 폐기
vault token revoke <token>
```

---

## 🔨 시크릿 생성 예제

### Harbor 시크릿 생성

```bash
# Docker config JSON 형식으로 생성
vault kv put secret/harbor/staging \
  .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{"username":"robot$staging","password":"your-token","auth":"base64-encoded"}}}'

vault kv put secret/harbor/production \
  .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{"username":"robot$production","password":"your-token","auth":"base64-encoded"}}}'
```

### 서버 환경 변수 생성

```bash
# Staging 환경
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  APP_ADMIN_EMAIL=admin@example.com \
  NODE_ENV=staging \
  DATABASE_URL=postgresql://user:pass@host:5432/db \
  AWS_ACCESS_KEY_ID=your-key \
  AWS_SECRET_ACCESS_KEY=your-secret \
  AWS_REGION=ap-northeast-2

# Production 환경
vault kv put secret/server/production \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  NODE_ENV=production \
  DATABASE_URL=postgresql://user:pass@prod-host:5432/db
```

### 헬퍼 스크립트 사용

```bash
# 대화형 시크릿 생성 도구
./scripts/create-secrets.sh
```

---

## 🛠️ 트러블슈팅

### "permission denied" 오류
```bash
# 현재 토큰의 정책 확인
vault token lookup | grep policies

# 정책이 허용하는 경로 확인
vault policy read esc-policy
```

### 연결 오류
```bash
# VAULT_ADDR 환경 변수 확인
echo $VAULT_ADDR

# 서버 상태 확인
vault status

# 네트워크 연결 테스트
curl $VAULT_ADDR/v1/sys/health
```

### 토큰 만료
```bash
# 토큰 TTL 확인
vault token lookup | grep ttl

# 토큰 갱신
vault token renew
```

### 시크릿 없음
```bash
# 경로 존재 확인
vault kv list secret/server

# 메타데이터 확인
vault kv metadata get secret/server/staging
```

---

## 📚 추가 문서

- **상세 CLI 설치 가이드**: [docs/vault-cli-installation.md](docs/vault-cli-installation.md)
- **외부 접근 방법**: [docs/external-access.md](docs/external-access.md)
- **Kubernetes 통합** (선택): [KUBERNETES.md](KUBERNETES.md)
- **전체 문서 인덱스**: [INDEX.md](INDEX.md)

## 🔒 보안 권장사항

1. **최소 권한 원칙**: 필요한 경로만 정책에 포함
2. **토큰 주기 관리**: 정기적인 토큰 교체 (3-6개월)
3. **토큰 저장**: Git에 절대 커밋하지 않음 (.gitignore 확인)
4. **로컬 환경**: 개발용 토큰과 프로덕션 토큰 분리

## 📖 참고 자료

- [OpenBao Documentation](https://openbao.org/docs/)
- [Vault CLI Commands](https://developer.hashicorp.com/vault/docs/commands)
