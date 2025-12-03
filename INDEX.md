# OpenBao 문서 인덱스

OpenBao 설정 및 사용을 위한 전체 문서 가이드입니다.

## 📚 빠른 시작 가이드

| 문서 | 설명 | 소요 시간 |
|------|------|-----------|
| [QUICKSTART.md](QUICKSTART.md) | ESC 설정 3단계 가이드 | 5분 |
| [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md) | 외부 접근 빠른 가이드 | 3분 |

## 🔧 설치 및 설정

### Vault CLI 설치
- [Vault CLI 설치 가이드](docs/vault-cli-installation.md) - 운영체제별 상세 설치 방법
- [자동 설치 스크립트](scripts/install-vault-cli.sh) - 원클릭 설치

### ESC (External Secrets Controller) 설정
- [README.md](README.md) - ESC 정책 및 토큰 설정 전체 가이드
- [정책 파일](policies/esc-policy.hcl) - ESC용 OpenBao 정책
- [설정 스크립트](scripts/setup-esc.sh) - 자동화된 ESC 설정
- [시크릿 생성 스크립트](scripts/create-secrets.sh) - 시크릿 생성 헬퍼

## 🌐 외부 접근 및 API 사용

### 외부 접근 방법
- [외부 접근 상세 가이드](docs/external-access.md) - 5가지 접근 방법 완전 가이드
- [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md) - 빠른 참조

### Bruno API 컬렉션
- [Bruno 컬렉션 README](bruno/README.md) - Bruno 사용 가이드
- [Bruno 컬렉션 파일들](bruno/OpenBao/) - 9개의 API 요청 템플릿

**컬렉션 내용:**
1. `01-health-check.bru` - 서버 상태 확인
2. `02-login.bru` - 로그인 및 토큰 획득
3. `03-token-lookup.bru` - 토큰 정보 조회
4. `04-list-secrets.bru` - 시크릿 목록
5. `05-read-secret-staging.bru` - Staging 시크릿
6. `06-read-secret-production.bru` - Production 시크릿
7. `07-write-secret.bru` - 시크릿 생성
8. `08-read-harbor-staging.bru` - Harbor 인증
9. `09-delete-secret.bru` - 시크릿 삭제

## 📋 참고 문서

### 정책 및 보안
- [ESC 정책](policies/esc-policy.hcl) - 읽기 전용 최소 권한 정책
- KV v1 vs v2 차이 - README의 KV 버전 설명 참고

### 현재 설정 정보
- **OpenBao URL**: `https://openbao.cocdev.co.kr`
- **Ingress**: nginx (Let's Encrypt)
- **시크릿 경로**:
  - `secret/data/server/staging`
  - `secret/data/server/production`
  - `secret/data/harbor/staging`
  - `secret/data/harbor/production`

## 🚀 사용 시나리오별 가이드

### 시나리오 1: 처음 시작하기
1. [Vault CLI 설치](docs/vault-cli-installation.md)
2. [외부 접근 빠른 가이드](EXTERNAL-ACCESS-QUICKSTART.md) - 연결 테스트
3. [ESC 설정 빠른 시작](QUICKSTART.md) - ESC 설정

### 시나리오 2: API 테스트하기
1. [Bruno 설치](https://www.usebruno.com/)
2. [Bruno 컬렉션 가이드](bruno/README.md) - 컬렉션 사용법
3. [외부 접근 가이드](docs/external-access.md) - API 엔드포인트

### 시나리오 3: Kubernetes에 ESC 배포
1. [ESC 정책 생성](scripts/setup-esc.sh) - 정책 및 토큰 생성
2. [시크릿 생성](scripts/create-secrets.sh) - OpenBao 시크릿 생성
3. [QUICKSTART.md](QUICKSTART.md) - Kubernetes Secret 생성
4. Helm 차트 배포 (README 참고)

### 시나리오 4: 로컬 개발
1. `kubectl port-forward -n openbao svc/openbao 8200:8200`
2. `export VAULT_ADDR=http://localhost:8200`
3. Bruno에서 "local" 환경 선택
4. 개발 진행

## 🛠️ 스크립트 및 도구

| 스크립트 | 용도 | 사용법 |
|----------|------|--------|
| [install-vault-cli.sh](scripts/install-vault-cli.sh) | Vault CLI 자동 설치 | `./scripts/install-vault-cli.sh` |
| [setup-esc.sh](scripts/setup-esc.sh) | ESC 정책 및 토큰 생성 | `./scripts/setup-esc.sh` |
| [create-secrets.sh](scripts/create-secrets.sh) | 시크릿 생성 헬퍼 | `./scripts/create-secrets.sh staging` |

## 🔍 트러블슈팅

문제 발생 시 참고할 문서:

| 문제 | 참고 문서 | 섹션 |
|------|----------|------|
| CLI 설치 오류 | [vault-cli-installation.md](docs/vault-cli-installation.md) | 트러블슈팅 |
| 연결 실패 | [external-access.md](docs/external-access.md) | 트러블슈팅 |
| 권한 오류 | [README.md](README.md) | 트러블슈팅 |
| ESC 동기화 실패 | [QUICKSTART.md](QUICKSTART.md) | 문제 해결 |
| Bruno 사용 문제 | [bruno/README.md](bruno/README.md) | 트러블슈팅 |

## 📞 빠른 참조

### 환경 변수
```bash
export VAULT_ADDR=https://openbao.cocdev.co.kr
export VAULT_TOKEN=your-token-here
```

### 기본 명령어
```bash
vault status                           # 상태 확인
vault login                            # 로그인
vault kv get secret/server/staging     # 시크릿 읽기
vault kv put secret/test key=value     # 시크릿 쓰기
vault kv list secret/server/           # 시크릿 목록
vault token lookup                     # 토큰 정보
```

### API 엔드포인트
```
POST   /v1/auth/userpass/login/:username
GET    /v1/secret/data/:path
POST   /v1/secret/data/:path
DELETE /v1/secret/data/:path
GET    /v1/sys/health
```

## 🔐 보안 체크리스트

- [ ] Vault CLI 설치됨
- [ ] HTTPS 연결 사용 (http:// 사용 금지)
- [ ] ESC 토큰은 읽기 전용 권한만
- [ ] 토큰이 Git에 커밋되지 않음
- [ ] Production/Staging 토큰 분리 (권장)
- [ ] 정기적인 토큰 교체 (3-6개월)
- [ ] 감사 로그 활성화

## 📖 추가 리소스

### 공식 문서
- [OpenBao Documentation](https://openbao.org/docs/)
- [Vault CLI Commands](https://www.vaultproject.io/docs/commands)
- [External Secrets Operator](https://external-secrets.io/)

### 관련 프로젝트
- [Bruno](https://www.usebruno.com/) - API 클라이언트
- [cert-manager](https://cert-manager.io/) - 인증서 관리
- [nginx-ingress](https://kubernetes.github.io/ingress-nginx/) - Ingress 컨트롤러

## 📝 문서 업데이트 이력

- 2024-12-04: 초기 문서 구조 생성
  - ESC 정책 및 스크립트
  - Bruno 컬렉션
  - 외부 접근 가이드
  - Vault CLI 설치 가이드
