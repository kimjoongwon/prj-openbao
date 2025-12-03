# Vault CLI 빠른 설치 가이드

## macOS에서 즉시 설치하기

### 방법 1: 자동 설치 스크립트 (가장 빠름) ⭐

```bash
cd /Users/wallykim/dev/prj-openbao
./scripts/install-vault-cli.sh
```

### 방법 2: Homebrew (권장)

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

### 방법 3: 수동 설치

```bash
# Intel Mac
curl -O https://releases.hashicorp.com/vault/1.18.3/vault_1.18.3_darwin_amd64.zip
unzip vault_1.18.3_darwin_amd64.zip
sudo mv vault /usr/local/bin/
rm vault_1.18.3_darwin_amd64.zip

# Apple Silicon (M1/M2/M3)
curl -O https://releases.hashicorp.com/vault/1.18.3/vault_1.18.3_darwin_arm64.zip
unzip vault_1.18.3_darwin_arm64.zip
sudo mv vault /usr/local/bin/
rm vault_1.18.3_darwin_arm64.zip
```

## 설치 확인

```bash
vault version
```

예상 출력:
```
Vault v1.18.3 (0123456789abcdef), built 2024-XX-XX
```

## 빠른 설정

### 1. OpenBao 연결 설정

```bash
# 환경 변수 설정
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 영구 설정 (zsh)
echo 'export VAULT_ADDR=https://openbao.cocdev.co.kr' >> ~/.zshrc
source ~/.zshrc

# 영구 설정 (bash)
echo 'export VAULT_ADDR=https://openbao.cocdev.co.kr' >> ~/.bash_profile
source ~/.bash_profile
```

### 2. 자동 완성 활성화 (선택사항)

```bash
vault -autocomplete-install

# zsh
source ~/.zshrc

# bash
source ~/.bash_profile
```

### 3. 연결 테스트

```bash
# 서버 상태 확인 (인증 불필요)
vault status

# 로그인
vault login

# 시크릿 읽기 테스트
vault kv get secret/server/staging
```

## 첫 사용 워크플로우

```bash
# 1. 서버 주소 설정
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 2. 서버 상태 확인
vault status

# 3. 로그인 (사용자명/비밀번호 입력)
vault login -method=userpass username=admin

# 4. 토큰 정보 확인
vault token lookup

# 5. 시크릿 목록 조회
vault kv list secret/server/

# 6. 시크릿 읽기
vault kv get secret/server/staging
vault kv get secret/server/production

# 7. Harbor 시크릿 읽기
vault kv get secret/harbor/staging
```

## 유용한 별칭 추가

`~/.zshrc` 또는 `~/.bash_profile`에 추가:

```bash
# OpenBao 별칭
alias vop='export VAULT_ADDR=https://openbao.cocdev.co.kr'
alias vlocal='export VAULT_ADDR=http://localhost:8200'
alias vs='vault status'
alias vl='vault login'
alias vget='vault kv get'
alias vput='vault kv put'
alias vlist='vault kv list'
alias vtoken='vault token lookup'
```

사용 예:
```bash
vop              # 프로덕션 서버 설정
vs               # 상태 확인
vl               # 로그인
vget secret/server/staging  # 시크릿 읽기
```

## 로컬 개발 환경

kubectl port-forward를 사용한 로컬 접근:

```bash
# 터미널 1: 포트 포워딩
kubectl port-forward -n openbao svc/openbao 8200:8200

# 터미널 2: 로컬 접근
export VAULT_ADDR=http://localhost:8200
vault login
```

## 문제 해결

### "command not found: vault"

```bash
# vault 경로 확인
which vault

# PATH에 추가
export PATH=$PATH:/usr/local/bin

# 영구 설정 (zsh)
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.zshrc
source ~/.zshrc
```

### "certificate verify failed"

```bash
# 인증서 검증 스킵 (테스트 환경만)
export VAULT_SKIP_VERIFY=true
```

### "connection refused"

```bash
# 서버 주소 확인
echo $VAULT_ADDR

# curl로 테스트
curl https://openbao.cocdev.co.kr/v1/sys/health

# 로컬 접근으로 전환
kubectl port-forward -n openbao svc/openbao 8200:8200
export VAULT_ADDR=http://localhost:8200
```

## 다음 단계

설치 완료 후:

1. ✅ **API 테스트**: [Bruno 컬렉션](bruno/README.md) 사용
2. ✅ **ESC 설정**: [QUICKSTART.md](QUICKSTART.md) 참고
3. ✅ **외부 접근**: [EXTERNAL-ACCESS-QUICKSTART.md](EXTERNAL-ACCESS-QUICKSTART.md) 참고

## 상세 문서

- [전체 설치 가이드](docs/vault-cli-installation.md) - 모든 운영체제 지원
- [문서 인덱스](INDEX.md) - 전체 문서 구조
