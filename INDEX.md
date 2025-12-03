# OpenBao 문서 인덱스

OpenBao CLI를 사용한 시크릿 관리를 위한 전체 문서 가이드입니다.

---

## 📚 빠른 시작 가이드

| 문서 | 설명 | 소요 시간 |
|------|------|-----------|
| [README.md](README.md) | 메인 가이드 (CLI 명령어 중심) | 10분 |
| [QUICKSTART.md](QUICKSTART.md) | CLI 빠른 시작 (3단계) | 5분 |
| [INSTALL-CLI.md](INSTALL-CLI.md) | CLI 빠른 설치 | 3분 |
| [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md) | 외부 접근 빠른 가이드 | 3분 |

---

## 🔧 설치 및 설정

### Vault CLI 설치
- **빠른 설치**: [INSTALL-CLI.md](INSTALL-CLI.md)
- **상세 설치**: [docs/vault-cli-installation.md](docs/vault-cli-installation.md) (OS별 가이드)
- **자동 설치**: [scripts/install-vault-cli.sh](scripts/install-vault-cli.sh)

### 정책 및 토큰 관리
- **정책 파일**: [policies/esc-policy.hcl](policies/esc-policy.hcl) (읽기 전용 정책)
- **자동 설정**: [scripts/setup-esc.sh](scripts/setup-esc.sh) (정책+토큰 생성)
- **시크릿 생성**: [scripts/create-secrets.sh](scripts/create-secrets.sh) (대화형 헬퍼)

---

## 🌐 외부 접근 및 사용

### 접근 방법
- **빠른 가이드**: [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md)
- **상세 가이드**: [docs/external-access.md](docs/external-access.md) (HTTPS, port-forward, curl 예제)

### 현재 설정 정보
- **OpenBao URL**: `https://openbao.cocdev.co.kr`
- **인증서**: Let's Encrypt (HTTPS)
- **시크릿 경로**:
  - Server: `secret/server/{staging|production|default}`
  - Harbor: `secret/harbor/{staging|production|development}`

---

## 🚀 사용 시나리오별 가이드

### 시나리오 1: 처음 시작하기 (신규 팀원)

**목표**: 5분 내에 CLI 설치 및 첫 시크릿 읽기

1. **CLI 설치** → [INSTALL-CLI.md](INSTALL-CLI.md)
   ```bash
   ./scripts/install-vault-cli.sh
   ```

2. **연결 및 로그인** → [QUICKSTART.md](QUICKSTART.md)
   ```bash
   export VAULT_ADDR=https://openbao.cocdev.co.kr
   vault login
   ```

3. **시크릿 조회**
   ```bash
   vault kv get secret/server/staging
   vault kv get secret/harbor/staging
   ```

**소요 시간**: 5분

---

### 시나리오 2: Harbor 시크릿 관리

**목표**: Harbor Registry 인증 정보 조회 및 업데이트

1. **현재 Harbor 시크릿 확인**
   ```bash
   vault kv list secret/harbor
   vault kv get secret/harbor/staging
   vault kv get secret/harbor/production
   ```

2. **Harbor 시크릿 업데이트**
   ```bash
   vault kv put secret/harbor/staging \
     .dockerconfigjson='{"auths":{"harbor.cocdev.co.kr":{...}}}'
   ```

3. **변경 이력 확인**
   ```bash
   vault kv metadata get secret/harbor/staging
   ```

**관련 문서**: [README.md - Harbor 시크릿 생성](README.md#-시크릿-생성-예제)

---

### 시나리오 3: 환경별 시크릿 관리 (Staging/Production)

**목표**: 환경별 서버 설정 관리

1. **Staging 환경 시크릿 생성/업데이트**
   ```bash
   vault kv put secret/server/staging \
     APP_PORT=3000 \
     APP_NAME=plate-api \
     NODE_ENV=staging \
     DATABASE_URL=postgresql://...
   ```

2. **Production 환경 시크릿 생성/업데이트**
   ```bash
   vault kv put secret/server/production \
     APP_PORT=3000 \
     NODE_ENV=production \
     DATABASE_URL=postgresql://...
   ```

3. **환경 간 비교**
   ```bash
   # 키 비교
   for env in staging production; do
     echo "$env:"
     vault kv get -field=APP_PORT secret/server/$env
   done
   ```

**관련 문서**: [QUICKSTART.md - 시크릿 생성](QUICKSTART.md#-시크릿-생성-예제)

---

### 시나리오 4: 로컬 개발 환경 설정

**목표**: 로컬에서 OpenBao 테스트

1. **Port-forward 시작**
   ```bash
   kubectl port-forward -n openbao svc/openbao 8200:8200 &
   ```

2. **로컬 주소로 연결**
   ```bash
   export VAULT_ADDR=http://localhost:8200
   vault login
   ```

3. **개발 진행**
   ```bash
   vault kv get secret/server/staging
   # ... 작업 진행
   ```

**관련 문서**: [docs/external-access.md - kubectl port-forward](docs/external-access.md#2-kubectl-port-forward-개발디버깅)

---

### 시나리오 5: 정책 및 토큰 생성 (관리자용)

**목표**: 팀원을 위한 새 토큰 생성

1. **정책 생성 (처음 한 번)**
   ```bash
   vault policy write esc-policy policies/esc-policy.hcl
   ```

2. **팀원용 토큰 생성**
   ```bash
   vault token create \
     -policy=esc-policy \
     -ttl=720h \
     -period=24h \
     -display-name=team-member-token
   ```

3. **자동화 스크립트 사용**
   ```bash
   ./scripts/setup-esc.sh
   ```

**관련 문서**: [README.md - 정책 및 토큰 관리](README.md#-정책-및-토큰-관리)

---

## 🛠️ 스크립트 및 도구

| 스크립트 | 용도 | 사용법 |
|----------|------|--------|
| [install-vault-cli.sh](scripts/install-vault-cli.sh) | Vault CLI 자동 설치 (macOS/Linux) | `./scripts/install-vault-cli.sh` |
| [setup-esc.sh](scripts/setup-esc.sh) | 정책 및 토큰 자동 생성 | `./scripts/setup-esc.sh` |
| [create-secrets.sh](scripts/create-secrets.sh) | 대화형 시크릿 생성 도구 | `./scripts/create-secrets.sh` |

---

## 🔍 트러블슈팅

문제 발생 시 참고할 문서:

| 문제 | 참고 문서 | 섹션 |
|------|----------|------|
| CLI 설치 오류 | [vault-cli-installation.md](docs/vault-cli-installation.md) | 트러블슈팅 |
| 연결 실패 ("connection refused") | [external-access.md](docs/external-access.md) | 트러블슈팅 |
| 권한 오류 ("permission denied") | [README.md](README.md) | 트러블슈팅 |
| 토큰 만료 | [QUICKSTART.md](QUICKSTART.md) | 문제 해결 |
| 시크릿 없음 | [QUICKSTART.md](QUICKSTART.md) | 문제 해결 |

---

## 📞 빠른 참조

### 환경 변수 설정
```bash
# 프로덕션 접근
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 로컬 개발 (port-forward 사용시)
export VAULT_ADDR=http://localhost:8200
```

### 기본 명령어
```bash
# 연결 및 인증
vault status                              # 서버 상태 확인
vault login                               # 로그인
vault token lookup                        # 토큰 정보 확인

# 시크릿 조회
vault kv list secret/server               # 목록 보기
vault kv get secret/server/staging        # 시크릿 읽기
vault kv get -format=json secret/...      # JSON 형식

# 시크릿 생성/수정
vault kv put secret/server/default \      # 새 시크릿
  APP_PORT=3000 APP_NAME=my-app
vault kv patch secret/server/staging \    # 부분 업데이트
  NEW_KEY=new_value

# 버전 관리
vault kv metadata get secret/server/...   # 변경 이력
vault kv get -version=2 secret/...        # 특정 버전
vault kv rollback -version=1 secret/...   # 롤백

# 시크릿 삭제
vault kv delete secret/test               # Soft delete
vault kv undelete -versions=1 secret/...  # 복구
vault kv destroy -versions=1 secret/...   # 영구 삭제
```

### curl을 사용한 API 호출
```bash
# Health check
curl https://openbao.cocdev.co.kr/v1/sys/health | jq

# 로그인
TOKEN=$(curl -s -X POST \
  https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin \
  -d '{"password":"pass"}' | jq -r '.auth.client_token')

# 시크릿 읽기
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging | jq
```

---

## 🔐 보안 체크리스트

- [ ] Vault CLI 설치 완료
- [ ] HTTPS 연결 사용 (프로덕션)
- [ ] 토큰이 Git에 커밋되지 않음
- [ ] Production/Staging 토큰 분리 (권장)
- [ ] 정기적인 토큰 교체 (3-6개월)
- [ ] 최소 권한 원칙 적용
- [ ] 민감한 정보는 안전한 채널로만 공유

---

## 📖 고급 주제

### Kubernetes 통합 (선택사항)
Kubernetes와 External Secrets Operator를 사용한 자동화된 시크릿 동기화가 필요한 경우:

- [KUBERNETES.md](KUBERNETES.md) - ESC 통합, Helm 배포, SecretStore 설정

**주의**: 이 문서는 선택적 고급 기능이며, 기본 CLI 작업에는 필요하지 않습니다.

---

## 📚 추가 리소스

### 공식 문서
- [OpenBao Documentation](https://openbao.org/docs/)
- [Vault CLI Commands](https://developer.hashicorp.com/vault/docs/commands)
- [Vault API Documentation](https://developer.hashicorp.com/vault/api-docs)

### 관련 기술
- [Kubernetes](https://kubernetes.io/)
- [External Secrets Operator](https://external-secrets.io/)
- [Docker](https://www.docker.com/)

---

## 📝 문서 업데이트 이력

- **2024-12-04**: CLI 중심으로 프로젝트 정리
  - Bruno API 컬렉션 제거 (CLI만 사용)
  - Kubernetes 관련 내용 KUBERNETES.md로 분리
  - 시나리오별 가이드 강화
  - CLI 명령어 예제 추가
