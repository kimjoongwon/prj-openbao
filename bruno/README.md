# OpenBao Bruno 컬렉션

OpenBao API를 테스트하기 위한 Bruno 컬렉션입니다.

## 설치

1. [Bruno](https://www.usebruno.com/) 다운로드 및 설치
2. Bruno 실행 후 "Open Collection" 선택
3. 이 디렉토리 선택: `/Users/wallykim/dev/prj-openbao/bruno`

## 환경 설정

### Production 환경
- **Base URL**: `https://openbao.cocdev.co.kr`
- **파일**: `environments/production.bru`

### Local 환경 (kubectl port-forward)
- **Base URL**: `http://localhost:8200`
- **파일**: `environments/local.bru`

### 환경 변수 설정 방법

1. Bruno에서 환경 선택 (좌측 상단)
2. 환경 파일 편집
3. `username`과 `password` 입력

```javascript
vars {
  base_url: https://openbao.cocdev.co.kr
  username: admin
  password: your-password-here
  token: // 로그인 후 자동 설정됨
}
```

## 사용 방법

### 1. 헬스 체크
```
01-health-check.bru
```
OpenBao 서버 상태 확인 (인증 불필요)

### 2. 로그인
```
02-login.bru
```
- 환경 변수의 `username`과 `password` 사용
- 성공 시 `token`이 자동으로 환경 변수에 저장됨
- 이후 모든 요청에서 이 토큰 사용

### 3. 토큰 정보 확인
```
03-token-lookup.bru
```
현재 토큰의 정책, TTL 등 확인

### 4. 시크릿 목록 조회
```
04-list-secrets.bru
```
`secret/server/` 경로의 시크릿 목록 확인

### 5. 시크릿 읽기
```
05-read-secret-staging.bru    # Staging 서버 환경 변수
06-read-secret-production.bru # Production 서버 환경 변수
08-read-harbor-staging.bru    # Harbor 인증 정보
```

### 6. 시크릿 쓰기
```
07-write-secret.bru
```
테스트용 시크릿 생성

### 7. 시크릿 삭제
```
09-delete-secret.bru
```
Soft delete (복구 가능)

## 요청 순서

처음 사용 시 권장 순서:

1. **Health Check** → 서버 상태 확인
2. **Login** → 토큰 획득
3. **Token Lookup** → 토큰 권한 확인
4. **List Secrets** → 사용 가능한 시크릿 확인
5. **Read Secret** → 실제 시크릿 값 읽기

## Pre-request & Post-response 스크립트

각 요청에는 자동화 스크립트가 포함되어 있습니다:

### Login 요청
```javascript
script:post-response {
  if (res.status === 200) {
    // 토큰을 환경 변수에 자동 저장
    bru.setEnvVar('token', res.body.auth.client_token);
    console.log('✅ 로그인 성공');
  }
}
```

### 모든 인증 요청
```javascript
headers {
  X-Vault-Token: {{token}}
}
```

## KV v2 경로 구조

OpenBao는 KV v2를 사용하므로 경로 구조가 다릅니다:

| 작업 | 경로 | 메서드 |
|------|------|--------|
| 읽기 | `/v1/secret/data/PATH` | GET |
| 쓰기 | `/v1/secret/data/PATH` | POST |
| 삭제 | `/v1/secret/data/PATH` | DELETE |
| 목록 | `/v1/secret/metadata/PATH?list=true` | GET |
| 메타 | `/v1/secret/metadata/PATH` | GET |

## 트러블슈팅

### 403 Permission Denied
- 토큰이 만료되었을 수 있음 → 다시 로그인
- 해당 경로에 대한 권한이 없음 → 정책 확인

### 404 Not Found
- 시크릿이 존재하지 않음
- 경로가 잘못됨 (KV v2 경로 확인)

### 인증서 오류 (Production)
```bash
# 인증서 확인
curl https://openbao.cocdev.co.kr/v1/sys/health
```

### Local 환경 연결 실패
```bash
# port-forward 실행 확인
kubectl port-forward -n openbao svc/openbao 8200:8200
```

## 추가 요청 생성

새로운 요청을 추가하려면:

1. Bruno에서 "New Request" 클릭
2. 요청 타입 선택 (HTTP/GraphQL/etc)
3. `.bru` 파일이 자동으로 생성됨
4. 요청 설정 및 스크립트 작성

### 템플릿
```javascript
meta {
  name: My Request
  type: http
  seq: 10
}

get {
  url: {{base_url}}/v1/some/path
}

headers {
  X-Vault-Token: {{token}}
}

docs {
  # 요청 설명

  여기에 문서 작성
}

script:post-response {
  if (res.status === 200) {
    console.log('✅ 성공');
  } else {
    console.log('❌ 실패');
  }
}
```

## 참고 자료

- [Bruno Documentation](https://docs.usebruno.com/)
- [OpenBao API Reference](https://openbao.org/api-docs/)
- [OpenBao CLI Reference](https://openbao.org/docs/commands/)
