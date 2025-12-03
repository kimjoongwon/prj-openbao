# OpenBao 외부 접근 빠른 가이드

## 🌐 현재 접근 가능한 주소

**프로덕션 URL**: `https://openbao.cocdev.co.kr`

✅ HTTPS (Let's Encrypt)
✅ 외부 접근 가능
✅ 인증서 자동 갱신

---

## 🚀 접근 방법

### 1. HTTPS 직접 접근 (권장) ✅

**프로덕션 환경에서 사용하는 방법입니다.**

```bash
# 환경 변수 설정
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 로그인
vault login

# 상태 확인
vault status

# 시크릿 읽기
vault kv get secret/server/staging
vault kv get secret/harbor/production
```

#### 영구 설정

Shell 설정 파일에 추가:

```bash
# ~/.zshrc 또는 ~/.bashrc에 추가
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 설정 적용
source ~/.zshrc
```

---

### 2. kubectl port-forward (로컬 개발)

**로컬 개발 및 테스트용입니다.**

```bash
# 터미널 1: 포트 포워딩 시작
kubectl port-forward -n openbao svc/openbao 8200:8200

# 터미널 2: 로컬 접근
export VAULT_ADDR=http://localhost:8200
vault login
vault kv get secret/server/staging
```

---

## 📊 접근 방법 비교

| 방법 | 용도 | 장점 | 단점 |
|------|------|------|------|
| **HTTPS (프로덕션)** | 일반 사용 | ✅ 안전<br>✅ 어디서나 접근 | ⚠️ 인터넷 필요 |
| **kubectl port-forward** | 로컬 개발 | ✅ 빠름<br>✅ 인증서 불필요<br>✅ 방화벽 우회 | ❌ 로컬만<br>❌ 세션 종료시 연결 끊김 |

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
# 목록 보기
vault kv list secret/server
vault kv list secret/harbor

# 특정 시크릿 읽기
vault kv get secret/server/staging
vault kv get secret/harbor/production

# JSON 형식으로 출력
vault kv get -format=json secret/server/staging
```

### 시크릿 생성/수정

```bash
# 새 시크릿 생성
vault kv put secret/server/default \
  APP_PORT=3000 \
  APP_NAME=my-app

# 부분 업데이트
vault kv patch secret/server/staging \
  NEW_KEY=new_value
```

---

## 🌐 curl을 사용한 API 호출

### Health Check

```bash
# 토큰 없이 서버 상태 확인
curl https://openbao.cocdev.co.kr/v1/sys/health | jq
```

### 로그인 및 토큰 획득

```bash
# 로그인하여 토큰 획득
TOKEN=$(curl -s -X POST \
  https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin \
  -d '{"password":"YOUR_PASSWORD"}' | jq -r '.auth.client_token')

echo "Token: $TOKEN"
```

### 시크릿 읽기

```bash
# 토큰으로 시크릿 읽기
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | jq

# 특정 필드만 추출
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | \
  jq -r '.data.data.APP_PORT'
```

### 시크릿 생성

```bash
# 새 시크릿 생성
curl -H "X-Vault-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{"data":{"APP_PORT":"3000","APP_NAME":"my-app"}}' \
  https://openbao.cocdev.co.kr/v1/secret/data/server/default
```

---

## 🔐 API 엔드포인트

### 인증
```
POST /v1/auth/userpass/login/:username
```

### 시크릿 (KV v2)
```
GET    /v1/secret/data/:path          # 읽기
POST   /v1/secret/data/:path          # 쓰기/수정
PATCH  /v1/secret/data/:path          # 부분 수정
DELETE /v1/secret/data/:path          # 삭제
GET    /v1/secret/metadata/:path      # 메타데이터
LIST   /v1/secret/metadata/:path      # 목록
```

### 토큰
```
GET  /v1/auth/token/lookup-self       # 토큰 정보
POST /v1/auth/token/renew-self        # 토큰 갱신
POST /v1/auth/token/revoke-self       # 토큰 폐기
```

### 시스템
```
GET /v1/sys/health                    # 헬스 체크
```

---

## 🛠️ 트러블슈팅

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

# 시스템 CA 인증서 업데이트
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install ca-certificates

# macOS는 자동 처리됨
```

### "permission denied" 오류

```bash
# 1. 토큰 정보 확인
vault token lookup

# 2. 정책 확인
vault token lookup | grep policies
vault policy read <policy-name>

# 3. 경로 접근 권한 테스트
vault kv get secret/server/staging
```

### 연결 타임아웃

```bash
# 1. 네트워크 연결 확인
ping openbao.cocdev.co.kr

# 2. 방화벽 확인
telnet openbao.cocdev.co.kr 443

# 3. port-forward 사용 (대안)
kubectl port-forward -n openbao svc/openbao 8200:8200
export VAULT_ADDR=http://localhost:8200
```

---

## 📚 추가 문서

- **상세 가이드**: [docs/external-access.md](docs/external-access.md)
- **빠른 시작**: [QUICKSTART.md](QUICKSTART.md)
- **CLI 설치**: [INSTALL-CLI.md](INSTALL-CLI.md)
- **메인 가이드**: [README.md](README.md)
- **Kubernetes 통합** (선택): [KUBERNETES.md](KUBERNETES.md)

---

## 🎯 다음 단계

1. ✅ HTTPS로 접근 확인: `vault status`
2. ✅ 로그인 테스트: `vault login`
3. ✅ 시크릿 읽기 테스트: `vault kv get secret/server/staging`
4. ✅ 시크릿 생성 테스트: `vault kv put secret/test key=value`
5. ✅ 정책 및 토큰 생성 (관리자): `./scripts/setup-esc.sh`

## 💡 팁

### 자주 사용하는 환경 변수

```bash
# ~/.zshrc 또는 ~/.bashrc에 추가
export VAULT_ADDR=https://openbao.cocdev.co.kr
export VAULT_FORMAT=json  # JSON 형식 기본 출력
```

### 여러 환경 관리

```bash
# 환경별 alias 설정
alias vault-prod='VAULT_ADDR=https://openbao.cocdev.co.kr vault'
alias vault-local='VAULT_ADDR=http://localhost:8200 vault'

# 사용 예
vault-prod kv get secret/server/production
vault-local kv get secret/server/staging
```
