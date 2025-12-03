# OpenBao 외부 접근 빠른 가이드

## 🌐 현재 접근 가능한 주소

**프로덕션 URL**: `https://openbao.cocdev.co.kr`

✅ HTTPS (Let's Encrypt)
✅ 외부 접근 가능
✅ 인증서 자동 갱신

## 🚀 빠른 시작

### 1. Vault CLI 사용

```bash
# 환경 변수 설정
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 로그인
vault login

# 상태 확인
vault status

# 시크릿 읽기
vault kv get secret/server/staging
```

### 2. Bruno API 클라이언트 사용

```bash
# Bruno 설치
# https://www.usebruno.com/

# 컬렉션 열기
# File → Open Collection
# 경로: /Users/wallykim/dev/prj-openbao/bruno

# 환경 설정 (Production 환경 선택)
# username과 password 입력

# 순서대로 실행:
1. Health Check (서버 확인)
2. Login (토큰 획득)
3. Read Secret (시크릿 읽기)
```

### 3. curl 사용

```bash
# 1. 로그인하여 토큰 획득
TOKEN=$(curl -s -X POST \
  https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin \
  -d '{"password":"YOUR_PASSWORD"}' | jq -r '.auth.client_token')

# 2. 시크릿 읽기
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | jq

# 3. 시크릿 쓰기
curl -H "X-Vault-Token: $TOKEN" \
  -X POST \
  -d '{"data":{"KEY":"VALUE"}}' \
  https://openbao.cocdev.co.kr/v1/secret/data/test/example
```

## 🏠 로컬 개발 환경

kubectl port-forward를 사용한 로컬 접근:

```bash
# 터미널 1: 포트 포워딩
kubectl port-forward -n openbao svc/openbao 8200:8200

# 터미널 2: 로컬 접근
export VAULT_ADDR=http://localhost:8200
vault login
vault kv get secret/server/staging
```

## 📊 접근 방법 비교

| 방법 | 용도 | 장점 | 단점 |
|------|------|------|------|
| **HTTPS (프로덕션)** | 일반 사용 | 안전, 어디서나 접근 | 인터넷 필요 |
| **kubectl port-forward** | 로컬 개발 | 빠름, 인증서 불필요 | 로컬만 |
| **NodePort** | 테스트 | 간단 | 보안 취약 |
| **LoadBalancer** | 클라우드 | 자동 부하분산 | 비용 발생 |

## 🔐 API 엔드포인트

### 인증
```
POST /v1/auth/userpass/login/:username
```

### 시크릿 (KV v2)
```
GET    /v1/secret/data/:path          # 읽기
POST   /v1/secret/data/:path          # 쓰기
DELETE /v1/secret/data/:path          # 삭제
GET    /v1/secret/metadata/:path      # 메타데이터
```

### 시스템
```
GET /v1/sys/health                    # 헬스 체크
GET /v1/auth/token/lookup-self        # 토큰 정보
POST /v1/auth/token/renew-self        # 토큰 갱신
```

## 🛠️ 트러블슈팅

### "connection refused" 오류
```bash
# 1. 서비스 상태 확인
kubectl get svc -n openbao

# 2. Ingress 확인
kubectl get ingress -n openbao

# 3. DNS 확인
nslookup openbao.cocdev.co.kr
```

### 인증서 오류
```bash
# 인증서 상태 확인
kubectl get certificate -n openbao

# cert-manager 로그
kubectl logs -n cert-manager deployment/cert-manager
```

### 403 Permission Denied
```bash
# 토큰 정보 확인
vault token lookup

# 정책 확인
vault token lookup | grep policies
```

## 📚 추가 문서

- 상세 가이드: [docs/external-access.md](docs/external-access.md)
- Bruno 컬렉션: [bruno/README.md](bruno/README.md)
- ESC 설정: [QUICKSTART.md](QUICKSTART.md)

## 🎯 다음 단계

1. ✅ HTTPS로 접근 확인
2. ✅ Bruno 컬렉션 설치
3. ✅ 로그인 테스트
4. ✅ 시크릿 읽기/쓰기 테스트
5. ✅ ESC 토큰 생성 (필요시)
