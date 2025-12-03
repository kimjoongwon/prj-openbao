# OpenBao 외부 접근 가이드

## 현재 프로덕션 설정

OpenBao는 다음 주소로 접근 가능합니다:

- **도메인**: `https://openbao.cocdev.co.kr`
- **인증서**: Let's Encrypt (HTTPS 보안 통신)

---

## 접근 방법

### 1. HTTPS를 통한 직접 접근 (권장) ✅

**프로덕션 환경에서 권장하는 방법입니다.**

#### Vault CLI 사용

```bash
# 환경 변수 설정
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 서버 상태 확인
vault status

# 로그인
vault login

# 시크릿 조회
vault kv get secret/server/staging
```

#### 영구 설정 (선택)

Shell 설정 파일에 추가하여 매번 입력하지 않도록 설정:

```bash
# ~/.zshrc 또는 ~/.bashrc에 추가
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 설정 적용
source ~/.zshrc
```

#### curl 사용

```bash
# 헬스 체크 (토큰 불필요)
curl https://openbao.cocdev.co.kr/v1/sys/health

# 로그인
TOKEN=$(curl -s -X POST \
  https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin \
  -d '{"password":"your-password"}' | jq -r '.auth.client_token')

# 시크릿 읽기
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | jq
```

---

### 2. kubectl port-forward (개발/디버깅)

**로컬 개발 환경에서 테스트용으로 사용합니다.**

```bash
# 포트 포워딩 시작 (백그라운드)
kubectl port-forward -n openbao svc/openbao 8200:8200 &

# 로컬 주소로 접근
export VAULT_ADDR=http://localhost:8200
vault status
vault login

# 시크릿 조회
vault kv get secret/server/staging
```

#### 다른 포트 사용

```bash
# 8200 포트가 사용 중인 경우
kubectl port-forward -n openbao svc/openbao 9200:8200

export VAULT_ADDR=http://localhost:9200
vault login
```

#### 장점
- ✅ 빠른 로컬 테스트
- ✅ 인증서 걱정 없음
- ✅ 방화벽 우회 가능

#### 단점
- ❌ 터미널 세션이 끊기면 연결 종료
- ❌ 한 번에 한 사용자만 가능
- ❌ 프로덕션 사용 부적합

---

## API 직접 호출 예제

### Health Check

```bash
# 토큰 불필요
curl https://openbao.cocdev.co.kr/v1/sys/health | jq
```

### 로그인 (사용자/비밀번호)

```bash
# POST /v1/auth/userpass/login/{username}
curl -X POST https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin \
  -H "Content-Type: application/json" \
  -d '{"password":"your-password"}' | jq

# 응답에서 client_token 추출
TOKEN=$(curl -s -X POST \
  https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin \
  -d '{"password":"your-password"}' | jq -r '.auth.client_token')

echo "Token: $TOKEN"
```

### 토큰 정보 조회

```bash
# GET /v1/auth/token/lookup-self
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/auth/token/lookup-self | jq
```

### 시크릿 목록

```bash
# LIST /v1/secret/metadata/server
curl -X LIST \
  -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/metadata/server | jq
```

### 시크릿 읽기

```bash
# GET /v1/secret/data/{path}
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | jq

# 특정 필드만 추출
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | \
  jq -r '.data.data.APP_PORT'
```

### 시크릿 생성/수정

```bash
# POST /v1/secret/data/{path}
curl -X POST \
  -H "X-Vault-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "APP_PORT": "3000",
      "APP_NAME": "plate-api",
      "NODE_ENV": "staging"
    }
  }' \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | jq
```

### 시크릿 삭제

```bash
# DELETE /v1/secret/data/{path}
curl -X DELETE \
  -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | jq
```

---

## 스크립트 예제

### 환경별 시크릿 조회 스크립트

```bash
#!/bin/bash
# check-secrets.sh

export VAULT_ADDR=https://openbao.cocdev.co.kr
ENVIRONMENTS=("staging" "production" "default")

echo "=== OpenBao Secrets Check ==="
echo

for env in "${ENVIRONMENTS[@]}"; do
  echo "[$env 환경]"

  # 서버 시크릿
  echo -n "  서버 시크릿: "
  if vault kv get secret/server/$env > /dev/null 2>&1; then
    echo "✅ 존재"
  else
    echo "❌ 없음"
  fi

  # Harbor 시크릿
  echo -n "  Harbor 시크릿: "
  if vault kv get secret/harbor/$env > /dev/null 2>&1; then
    echo "✅ 존재"
  else
    echo "❌ 없음"
  fi

  echo
done
```

### 시크릿 백업 스크립트

```bash
#!/bin/bash
# backup-secrets.sh

export VAULT_ADDR=https://openbao.cocdev.co.kr
BACKUP_DIR="./backups/$(date +%Y%m%d)"

mkdir -p "$BACKUP_DIR"

echo "=== OpenBao Secrets Backup ==="
echo "Backup directory: $BACKUP_DIR"
echo

# 서버 시크릿 백업
for env in staging production default; do
  echo "Backing up server/$env..."
  vault kv get -format=json secret/server/$env > "$BACKUP_DIR/server-$env.json"
done

# Harbor 시크릿 백업
for env in staging production development; do
  echo "Backing up harbor/$env..."
  vault kv get -format=json secret/harbor/$env > "$BACKUP_DIR/harbor-$env.json"
done

echo "✅ Backup completed!"
ls -lh "$BACKUP_DIR"
```

---

## 보안 권장사항

### 1. HTTPS 사용 (현재 설정)
✅ Let's Encrypt 인증서 사용 중
✅ 자동 갱신 설정됨

### 2. 토큰 관리
- ❌ 토큰을 Git에 커밋하지 마세요
- ❌ 토큰을 평문으로 저장하지 마세요
- ✅ 정기적으로 토큰 교체 (3-6개월)
- ✅ 불필요한 토큰은 즉시 폐기

```bash
# 토큰 폐기
vault token revoke <token>
```

### 3. 네트워크 보안
- ✅ HTTPS만 사용 (HTTP 비활성화)
- ✅ 인증서 유효성 검증
- ⚠️ 공용 WiFi에서 사용 주의

### 4. 접근 제어
- ✅ 최소 권한 원칙 적용
- ✅ 환경별 토큰 분리 (staging/production)
- ✅ 팀원별 개별 계정 사용

---

## 트러블슈팅

### "connection refused" 오류

```bash
# 1. 서버 주소 확인
echo $VAULT_ADDR

# 2. 네트워크 연결 테스트
curl https://openbao.cocdev.co.kr/v1/sys/health

# 3. DNS 확인
nslookup openbao.cocdev.co.kr
```

### "certificate verify failed" 오류

```bash
# 인증서 정보 확인
openssl s_client -connect openbao.cocdev.co.kr:443 -showcerts

# 시스템 CA 인증서 업데이트 (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install ca-certificates

# macOS
# 시스템 키체인에서 자동 처리됨
```

### "permission denied" 오류

```bash
# 1. 토큰 확인
vault token lookup

# 2. 정책 확인
vault token lookup | grep policies
vault policy read <policy-name>

# 3. 경로 접근 권한 확인
vault kv get secret/server/staging
```

### 연결 타임아웃

```bash
# 1. 네트워크 연결 확인
ping openbao.cocdev.co.kr

# 2. 방화벽 확인
telnet openbao.cocdev.co.kr 443

# 3. kubectl port-forward 사용 (대안)
kubectl port-forward -n openbao svc/openbao 8200:8200
export VAULT_ADDR=http://localhost:8200
```

---

## 추가 문서

- **빠른 시작**: [../QUICKSTART.md](../QUICKSTART.md)
- **CLI 설치**: [../INSTALL-CLI.md](../INSTALL-CLI.md)
- **외부 접근 요약**: [../EXTERNAL-ACCESS-QUICKSTART.md](../EXTERNAL-ACCESS-QUICKSTART.md)
- **메인 가이드**: [../README.md](../README.md)
- **Kubernetes 통합** (선택): [../KUBERNETES.md](../KUBERNETES.md)

## 참고 자료

- [OpenBao Documentation](https://openbao.org/docs/)
- [Vault API Documentation](https://developer.hashicorp.com/vault/api-docs)
- [Vault CLI Commands](https://developer.hashicorp.com/vault/docs/commands)
