# OpenBao 관리 프로젝트

OpenBao 시크릿 관리를 위한 정책, 스크립트, API 컬렉션 및 문서 저장소입니다.

## 📋 프로젝트 구조

```
prj-openbao/
├── 📚 문서
│   ├── INDEX.md                           # 전체 문서 인덱스
│   ├── QUICKSTART.md                      # ESC 3단계 설정
│   ├── INSTALL-CLI.md                     # Vault CLI 빠른 설치
│   ├── EXTERNAL-ACCESS-QUICKSTART.md      # 외부 접근 가이드
│   └── docs/
│       ├── vault-cli-installation.md      # CLI 상세 설치
│       └── external-access.md             # 외부 접근 상세
│
├── 🔐 정책 및 스크립트
│   ├── policies/
│   │   └── esc-policy.hcl                # ESC 읽기 전용 정책
│   └── scripts/
│       ├── install-vault-cli.sh          # CLI 자동 설치
│       ├── setup-esc.sh                  # ESC 설정 자동화
│       └── create-secrets.sh             # 시크릿 생성 헬퍼
│
└── 🎨 API 컬렉션
    └── bruno/
        ├── environments/                  # 환경 설정
        └── OpenBao/                       # 9개 API 요청
```

## 🚀 빠른 시작

### 1. Vault CLI 설치
```bash
./scripts/install-vault-cli.sh
```

### 2. OpenBao 연결
```bash
export VAULT_ADDR=https://openbao.cocdev.co.kr
vault status
vault login
```

### 3. 문서 참고
- **CLI 설치**: [INSTALL-CLI.md](INSTALL-CLI.md)
- **외부 접근**: [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md)
- **ESC 설정**: [QUICKSTART.md](QUICKSTART.md)
- **전체 문서**: [INDEX.md](INDEX.md)

---

## ESC (External Secrets Controller) 설정

### 개요
External Secrets Operator가 OpenBao의 시크릿을 읽을 수 있도록 정책과 토큰을 설정합니다.

### 사전 요구사항
- OpenBao가 실행 중이어야 함
- Root 또는 관리자 권한의 토큰으로 로그인되어 있어야 함
- `vault` CLI 도구 설치 → [설치 가이드](docs/vault-cli-installation.md)

### 빠른 시작

```bash
# 1. OpenBao에 로그인
export VAULT_ADDR=http://your-openbao-address:8200
vault login

# 2. 설정 스크립트 실행
cd /Users/wallykim/dev/prj-openbao
chmod +x scripts/setup-esc.sh
./scripts/setup-esc.sh

# 3. 출력된 토큰을 복사하여 Kubernetes Secret 생성
```

### 수동 설정

#### 1. 정책 생성

```bash
# 정책 파일 적용
vault policy write esc-policy policies/esc-policy.hcl

# 정책 확인
vault policy read esc-policy
```

#### 2. 토큰 생성

```bash
# 기본 토큰 (30일, 자동 갱신)
vault token create \
  -policy=esc-policy \
  -ttl=720h \
  -period=24h \
  -renewable=true \
  -display-name=esc-token

# 무제한 TTL 토큰 (프로덕션 비권장)
vault token create \
  -policy=esc-policy \
  -no-default-policy \
  -period=24h \
  -display-name=esc-token-prod
```

#### 3. Kubernetes Secret 생성

**Staging 환경:**
```bash
kubectl create secret generic openbao-token \
  --from-literal=token=<생성된_토큰> \
  --namespace=external-secrets-stg
```

**Production 환경:**
```bash
kubectl create secret generic openbao-token \
  --from-literal=token=<생성된_토큰> \
  --namespace=external-secrets-prod
```

### 현재 정책이 허용하는 경로

ESC 정책은 다음 OpenBao 경로에 대한 읽기 권한을 부여합니다:

**서버 환경 변수**:
- `secret/data/server/staging`
- `secret/data/server/production`
- `secret/data/server/default`

**Harbor Registry 인증**:
- `secret/data/harbor/staging`
- `secret/data/harbor/production`
- `secret/data/harbor/development`

### OpenBao에 시크릿 생성

정책 적용 후, 실제 시크릿을 OpenBao에 생성해야 합니다:

```bash
# 1. Staging 서버 환경 변수 생성
vault kv put secret/server/staging \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  APP_ADMIN_EMAIL=admin@example.com \
  API_PREFIX=/api \
  APP_FALLBACK_LANGUAGE=ko \
  APP_HEADER_LANGUAGE=x-custom-lang \
  FRONTEND_DOMAIN=https://staging.example.com \
  BACKEND_DOMAIN=https://api-staging.example.com \
  NODE_ENV=staging \
  AWS_ACCESS_KEY_ID=your-key \
  AWS_SECRET_ACCESS_KEY=your-secret \
  AWS_REGION=ap-northeast-2 \
  AWS_S3_BUCKET_NAME=your-bucket \
  SMTP_HOST=smtp.example.com \
  SMTP_PORT=587 \
  SMTP_USERNAME=user \
  SMTP_PASSWORD=pass \
  SMTP_SENDER=noreply@example.com \
  AUTH_JWT_SECRET=your-jwt-secret \
  AUTH_JWT_TOKEN_EXPIRES_IN=3600 \
  AUTH_JWT_TOKEN_REFRESH_IN=86400 \
  AUTH_JWT_SALT_ROUNDS=10 \
  CORS_ENABLED=true \
  DATABASE_URL=postgresql://user:pass@host:5432/db \
  DIRECT_URL=postgresql://user:pass@host:5432/db

# 2. Production 서버 환경 변수 생성
vault kv put secret/server/production \
  APP_PORT=3000 \
  APP_NAME=plate-api \
  # ... (production 값들)

# 3. Staging Harbor 인증 생성
vault kv put secret/harbor/staging \
  .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{"username":"robot$staging","password":"your-token","auth":"base64-encoded"}}}'

# 4. Production Harbor 인증 생성
vault kv put secret/harbor/production \
  .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{"username":"robot$production","password":"your-token","auth":"base64-encoded"}}}'
```

### 정책 커스터마이징

`policies/esc-policy.hcl` 파일을 수정하여 접근 경로를 추가/제거할 수 있습니다:

```hcl
# 새로운 애플리케이션별 경로 추가 (현재는 주석 처리됨)
path "secret/data/plate-api/*" {
  capabilities = ["read", "list"]
}

path "secret/metadata/plate-api/*" {
  capabilities = ["read", "list"]
}
```

수정 후 정책 재적용:
```bash
vault policy write esc-policy policies/esc-policy.hcl
```

### 토큰 관리

#### 토큰 정보 확인
```bash
vault token lookup <token>
```

#### 토큰 갱신
```bash
vault token renew <token>
```

#### 토큰 폐기
```bash
vault token revoke <token>
```

#### 모든 ESC 토큰 조회
```bash
vault list auth/token/accessors
vault token lookup -accessor <accessor_id>
```

### 보안 권장사항

1. **최소 권한 원칙**: 필요한 경로만 정책에 포함
2. **토큰 주기 관리**:
   - `period` 설정으로 자동 갱신 활성화
   - 정기적인 토큰 교체 (3-6개월)
3. **토큰 저장**:
   - Kubernetes Secret에만 저장
   - Git에 커밋하지 않음 (.gitignore 확인)
4. **감사 로그**: OpenBao 감사 로그 활성화
   ```bash
   vault audit enable file file_path=/var/log/vault/audit.log
   ```

### 트러블슈팅

#### "permission denied" 오류
- 정책에 해당 경로가 포함되어 있는지 확인
- 토큰이 올바른 정책과 연결되어 있는지 확인
  ```bash
  vault token lookup | grep policies
  ```

#### 토큰 만료
- `period` 설정이 있다면 자동 갱신되어야 함
- ESC가 자동으로 갱신하도록 설정되어 있는지 확인

#### 경로 접근 불가
```bash
# 토큰으로 직접 테스트
VAULT_TOKEN=<esc_token> vault kv get secret/app/test
```

### SecretStore 설정 예시

**ClusterSecretStore (권장):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: openbao-backend
spec:
  provider:
    vault:
      server: "http://openbao.openbao-system.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: "openbao-token"
          namespace: "external-secrets-stg"
          key: "token"
```

**ExternalSecret 예시:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: plate-api-secrets
  namespace: plate-api-stg
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: openbao-backend
    kind: ClusterSecretStore
  target:
    name: plate-api-secrets
    creationPolicy: Owner
  data:
  - secretKey: DATABASE_URL
    remoteRef:
      key: secret/plate-api/database
      property: url
```

### 참고 자료
- [OpenBao Documentation](https://openbao.org/docs/)
- [External Secrets Operator - Vault Provider](https://external-secrets.io/latest/provider/hashicorp-vault/)
