# 토큰 무효화 빠른 가이드

## 🚀 Root 제외 모든 토큰 무효화 (원클릭)

### 자동 스크립트 사용 (권장)

```bash
cd ~/dev/prj-openbao
./scripts/revoke-non-root-tokens.sh
```

**기능**:
- ✅ Root 권한 자동 확인
- ✅ Root 토큰 자동 보존
- ✅ 각 토큰 정보 표시 (이름, 정책, TTL)
- ✅ 처리 결과 요약 출력

## 📝 수동 방법

### 1. 특정 토큰 무효화

```bash
# 토큰으로 무효화
vault token revoke hvs.XXXXX

# Accessor로 무효화
vault token revoke -accessor hmac-XXXXX
```

### 2. 모든 토큰 목록 확인

```bash
# Accessor 목록
vault list auth/token/accessors

# 상세 정보 포함
vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  echo "Accessor: $accessor"
  vault token lookup -accessor $accessor | grep -E "display_name|policies|ttl"
  echo "---"
done
```

### 3. 특정 정책의 토큰만 무효화

```bash
# ESC 정책 토큰만 무효화
vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  POLICIES=$(vault token lookup -accessor $accessor 2>/dev/null | grep "^policies" | awk '{print $2}')

  if echo "$POLICIES" | grep -q "esc-policy" && ! echo "$POLICIES" | grep -q "root"; then
    echo "Revoking ESC token: $accessor"
    vault token revoke -accessor $accessor
  fi
done
```

### 4. 특정 인증 방법의 토큰 무효화

```bash
# Userpass로 생성된 모든 토큰
vault token revoke -mode path auth/userpass

# Token 인증으로 생성된 토큰
vault token revoke -mode path auth/token
```

## 🔍 토큰 조회

### 현재 토큰 확인

```bash
vault token lookup
```

### 특정 토큰 확인

```bash
vault token lookup <token>
vault token lookup -accessor <accessor>
```

### Root 토큰 확인

```bash
vault token lookup | grep "policies.*root"
```

## ⚠️ 주의사항

### Root 토큰 보존

스크립트는 자동으로 Root 정책을 가진 토큰을 건너뜁니다:

```bash
# 이 토큰은 무효화되지 않습니다
policies: [root default]
policies: [root]
```

### 현재 사용 중인 토큰

자신의 토큰을 무효화하면 즉시 연결이 끊어집니다:

```bash
# 로그아웃
vault token revoke -self
```

### 무효화 후 재로그인

```bash
# Root 토큰으로 다시 로그인
vault login

# 또는 Userpass
vault login -method=userpass username=admin
```

## 📊 실행 예시

```bash
$ ./scripts/revoke-non-root-tokens.sh

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🗑️  Root 제외 모든 토큰 무효화
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 Step 1: Root 권한 확인
✅ Root 권한 확인됨

⚠️  경고: Root 토큰을 제외한 모든 토큰이 무효화됩니다!

계속하시겠습니까? (yes/no): yes

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 Step 2: 모든 토큰 조회 중...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 총 5 개의 토큰 발견

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🗑️  Step 3: 토큰 무효화 중...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⏭️  [SKIP] Root 토큰: token

🗑️  [DELETE] esc-token
    ├─ Policies: [default esc-policy]
    ├─ TTL: 719h59m30s
    └─ Created: 2024-12-03T10:00:00Z

🗑️  [DELETE] userpass-admin
    ├─ Policies: [default admin-policy]
    ├─ TTL: 23h59m45s
    └─ Created: 2024-12-04T09:00:00Z

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 무효화 완료
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

총 토큰 수:        5
무효화됨:          4
건너뜀 (Root):     1
실패:              0

✅ 4 개의 토큰이 무효화되었습니다.
⏭️  1 개의 Root 토큰이 보존되었습니다.
```

## 🔐 보안 권장사항

### 1. 정기적인 토큰 정리

```bash
# 월간 토큰 감사 및 정리
0 0 1 * * /path/to/revoke-non-root-tokens.sh
```

### 2. ESC 토큰 재생성

```bash
# 기존 ESC 토큰 무효화 후 재생성
./scripts/setup-esc.sh
```

### 3. Root 토큰 사용 최소화

```bash
# 작업 완료 후 즉시 Root 로그아웃
vault login <root-token>
# 필요한 작업 수행
vault token revoke -self
```

## 📚 추가 정보

- **상세 가이드**: [docs/token-management.md](docs/token-management.md)
- **ESC 설정**: [QUICKSTART.md](QUICKSTART.md)
- **문서 인덱스**: [INDEX.md](INDEX.md)

## 🛠️ 트러블슈팅

### "permission denied" 오류

```bash
# Root 권한 확인
vault token lookup | grep policies

# Root로 재로그인
vault login
```

### 스크립트 실행 권한 오류

```bash
chmod +x scripts/revoke-non-root-tokens.sh
```

### jq 명령어 없음

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt install jq
```
