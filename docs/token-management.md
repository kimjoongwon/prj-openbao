# OpenBao 토큰 관리 가이드

## 토큰 무효화 (Revoke)

### 1. 특정 토큰 무효화

```bash
# 토큰으로 직접 무효화
vault token revoke <token>

# Accessor로 무효화
vault token revoke -accessor <accessor>
```

### 2. 현재 토큰 무효화 (로그아웃)

```bash
vault token revoke -self
```

### 3. Root 토큰 제외하고 모든 토큰 무효화

#### 방법 1: Accessor 목록으로 무효화 (권장)

```bash
#!/bin/bash
# revoke-non-root-tokens.sh

# 현재 토큰 확인 (root인지 확인)
CURRENT_TOKEN=$(cat ~/.vault-token)
vault token lookup $CURRENT_TOKEN | grep -q "policies.*root" || {
  echo "❌ Root 권한이 필요합니다"
  exit 1
}

echo "🔍 모든 토큰 Accessor 조회 중..."

# 모든 토큰 accessor 목록 가져오기
ACCESSORS=$(vault list -format=json auth/token/accessors | jq -r '.[]')

echo "📋 총 $(echo "$ACCESSORS" | wc -l) 개의 토큰 발견"
echo ""

REVOKED=0
SKIPPED=0

for accessor in $ACCESSORS; do
  # Accessor로 토큰 정보 조회
  TOKEN_INFO=$(vault token lookup -accessor $accessor 2>/dev/null)

  if [ $? -ne 0 ]; then
    echo "⚠️  Accessor $accessor 조회 실패 (이미 삭제됨?)"
    continue
  fi

  # 정책 확인
  POLICIES=$(echo "$TOKEN_INFO" | grep "policies" | awk '{print $2}')
  DISPLAY_NAME=$(echo "$TOKEN_INFO" | grep "display_name" | awk '{print $2}')

  # Root 정책 포함 여부 확인
  if echo "$POLICIES" | grep -q "root"; then
    echo "⏭️  Root 토큰 건너뜀: $DISPLAY_NAME (accessor: ${accessor:0:8}...)"
    ((SKIPPED++))
  else
    echo "🗑️  토큰 무효화: $DISPLAY_NAME (accessor: ${accessor:0:8}...)"
    vault token revoke -accessor $accessor
    ((REVOKED++))
  fi
done

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ 완료: $REVOKED 개 무효화, $SKIPPED 개 건너뜀 (root)"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

#### 방법 2: 특정 인증 방법의 토큰만 무효화

```bash
# Userpass로 생성된 모든 토큰 무효화
vault token revoke -mode path auth/userpass

# Token 인증 방법으로 생성된 토큰 무효화
vault token revoke -mode path auth/token
```

#### 방법 3: 특정 정책의 토큰만 무효화

```bash
# esc-policy를 가진 모든 토큰 조회 및 무효화
vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  POLICIES=$(vault token lookup -accessor $accessor 2>/dev/null | grep policies | awk '{print $2}')

  if echo "$POLICIES" | grep -q "esc-policy" && ! echo "$POLICIES" | grep -q "root"; then
    echo "Revoking token with accessor: $accessor"
    vault token revoke -accessor $accessor
  fi
done
```

### 4. 만료된 토큰 정리

OpenBao는 자동으로 만료된 토큰을 정리하지만, 수동으로 확인할 수 있습니다:

```bash
# 모든 토큰의 TTL 확인
vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  echo "Accessor: $accessor"
  vault token lookup -accessor $accessor | grep -E "display_name|ttl|creation_time"
  echo ""
done
```

### 5. 모든 토큰 무효화 (Root 포함, 신중히!)

```bash
# ⚠️ 매우 위험! 모든 토큰이 무효화됩니다
vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  vault token revoke -accessor $accessor
done
```

## 토큰 조회 및 관리

### 현재 토큰 정보 확인

```bash
vault token lookup
```

### 특정 토큰 정보 확인

```bash
# 토큰으로 조회
vault token lookup <token>

# Accessor로 조회
vault token lookup -accessor <accessor>
```

### 모든 토큰 목록 조회

```bash
# Accessor 목록
vault list auth/token/accessors

# JSON 형식
vault list -format=json auth/token/accessors
```

### 토큰 갱신

```bash
# 현재 토큰 갱신
vault token renew

# 특정 토큰 갱신
vault token renew <token>

# 특정 TTL로 갱신
vault token renew -increment=1h <token>
```

## Root 토큰 관리

### Root 토큰 생성 (분실 시)

```bash
# 1. Root 토큰 생성 시작 (Unseal Key 필요)
vault operator generate-root -init

# 출력 예시:
# Nonce: 2dbd10f1-8528-6246-09e7-82b25b8afa63
# OTP: 8GU6Tz1k9wJ8ZbXj+ZKx+1nYw=

# 2. Unseal Key로 인증 (threshold만큼 반복)
vault operator generate-root -nonce=<nonce>
# Unseal Key 입력

# 3. Encoded Token 받음
# Encoded Token: DiWH/cG+9UAIY1aZN3Z7tA==

# 4. 디코딩
vault operator generate-root \
  -decode=<encoded-token> \
  -otp=<otp>
```

### Root 토큰 무효화 (사용 후)

```bash
# 특정 root 토큰 무효화
vault token revoke <root-token>

# 현재 root 토큰 무효화 (로그아웃)
vault token revoke -self
```

## 실용적인 토큰 정리 시나리오

### 시나리오 1: 정기적인 토큰 정리

```bash
#!/bin/bash
# cleanup-old-tokens.sh

echo "🧹 30일 이상 된 토큰 정리"

vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  TOKEN_INFO=$(vault token lookup -accessor $accessor 2>/dev/null)

  if [ $? -ne 0 ]; then
    continue
  fi

  # Root 토큰은 건너뛰기
  if echo "$TOKEN_INFO" | grep -q "policies.*root"; then
    continue
  fi

  # 생성 시간 확인 (30일 이상)
  CREATION_TIME=$(echo "$TOKEN_INFO" | grep "creation_time" | awk '{print $2}')
  CREATION_TIMESTAMP=$(date -j -f "%Y-%m-%dT%H:%M:%S" "${CREATION_TIME%.*}" +%s 2>/dev/null)
  NOW=$(date +%s)
  AGE_DAYS=$(( ($NOW - $CREATION_TIMESTAMP) / 86400 ))

  if [ $AGE_DAYS -gt 30 ]; then
    DISPLAY_NAME=$(echo "$TOKEN_INFO" | grep "display_name" | awk '{print $2}')
    echo "🗑️  $AGE_DAYS 일 된 토큰 무효화: $DISPLAY_NAME"
    vault token revoke -accessor $accessor
  fi
done
```

### 시나리오 2: 특정 사용자의 모든 토큰 무효화

```bash
#!/bin/bash
# revoke-user-tokens.sh

USERNAME="$1"

if [ -z "$USERNAME" ]; then
  echo "사용법: $0 <username>"
  exit 1
fi

echo "🔍 사용자 '$USERNAME'의 토큰 검색 중..."

vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  TOKEN_INFO=$(vault token lookup -accessor $accessor 2>/dev/null)

  if [ $? -ne 0 ]; then
    continue
  fi

  DISPLAY_NAME=$(echo "$TOKEN_INFO" | grep "display_name" | awk '{print $2}')

  if echo "$DISPLAY_NAME" | grep -q "userpass-$USERNAME"; then
    echo "🗑️  토큰 무효화: $DISPLAY_NAME"
    vault token revoke -accessor $accessor
  fi
done
```

### 시나리오 3: ESC 토큰만 갱신

```bash
#!/bin/bash
# renew-esc-tokens.sh

echo "🔄 ESC 토큰 갱신 중..."

vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  TOKEN_INFO=$(vault token lookup -accessor $accessor 2>/dev/null)

  if [ $? -ne 0 ]; then
    continue
  fi

  POLICIES=$(echo "$TOKEN_INFO" | grep "policies" | awk '{print $2}')

  if echo "$POLICIES" | grep -q "esc-policy"; then
    DISPLAY_NAME=$(echo "$TOKEN_INFO" | grep "display_name" | awk '{print $2}')
    echo "♻️  ESC 토큰 갱신: $DISPLAY_NAME"

    # Accessor로는 갱신 불가, 토큰이 필요
    # 대신 정보만 표시
    echo "  - TTL: $(echo "$TOKEN_INFO" | grep "ttl" | awk '{print $2}')"
  fi
done
```

## 보안 권장사항

### 1. Root 토큰 사용 최소화

```bash
# Root 토큰 사용 직후 무효화
vault login <root-token>
# 필요한 작업 수행
vault token revoke -self
```

### 2. 정기적인 토큰 감사

```bash
# 월간 토큰 감사
vault list -format=json auth/token/accessors | jq -r '.[]' | while read accessor; do
  vault token lookup -accessor $accessor 2>/dev/null | \
    grep -E "display_name|policies|creation_time|ttl"
  echo "---"
done > token-audit-$(date +%Y%m%d).log
```

### 3. 토큰 생성 제한

```bash
# 토큰 생성 정책에 제한 추가
# policy.hcl
path "auth/token/create" {
  capabilities = ["create", "update"]
  allowed_policies = ["esc-policy", "app-policy"]
  max_ttl = "720h"
}
```

### 4. 자동 만료 설정

```bash
# TTL이 있는 토큰만 생성
vault token create \
  -policy=esc-policy \
  -ttl=24h \
  -period=24h \
  -renewable=true
```

## 트러블슈팅

### "permission denied" 오류

```bash
# Root 권한 확인
vault token lookup | grep policies

# Root 권한이 없다면 root 토큰으로 다시 로그인
vault login
```

### Accessor로 조회 실패

```bash
# 토큰이 이미 삭제되었을 수 있음
# 에러 무시하고 계속 진행
vault token lookup -accessor $accessor 2>/dev/null || echo "Token not found"
```

### 대량 무효화 시 성능

```bash
# 병렬 처리로 속도 향상
vault list -format=json auth/token/accessors | jq -r '.[]' | \
  xargs -P 10 -I {} sh -c 'vault token revoke -accessor {}'
```

## 참고 자료

- [Vault Token Commands](https://www.vaultproject.io/docs/commands/token)
- [Token Auth Method](https://www.vaultproject.io/docs/auth/token)
- [Token Lifecycle](https://www.vaultproject.io/docs/concepts/tokens)
