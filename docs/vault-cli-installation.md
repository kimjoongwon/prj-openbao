# Vault CLI 설치 가이드

OpenBao는 HashiCorp Vault의 포크이므로 Vault CLI를 그대로 사용할 수 있습니다.

## macOS 설치

### 방법 1: Homebrew (권장)

```bash
# Homebrew로 설치
brew tap hashicorp/tap
brew install hashicorp/tap/vault

# 설치 확인
vault version
```

### 방법 2: 바이너리 다운로드

```bash
# 1. 다운로드
curl -O https://releases.hashicorp.com/vault/1.18.3/vault_1.18.3_darwin_amd64.zip

# Apple Silicon (M1/M2/M3)
curl -O https://releases.hashicorp.com/vault/1.18.3/vault_1.18.3_darwin_arm64.zip

# 2. 압축 해제
unzip vault_1.18.3_darwin_*.zip

# 3. 실행 권한 부여
chmod +x vault

# 4. PATH에 추가
sudo mv vault /usr/local/bin/

# 5. 확인
vault version
```

## Linux 설치

### Ubuntu/Debian

```bash
# 1. GPG 키 추가
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# 2. 리포지토리 추가
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# 3. 설치
sudo apt update && sudo apt install vault

# 4. 확인
vault version
```

### CentOS/RHEL/Fedora

```bash
# 1. 리포지토리 추가
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# 2. 설치
sudo yum -y install vault

# 3. 확인
vault version
```

### 바이너리 설치 (모든 Linux)

```bash
# 1. 다운로드
wget https://releases.hashicorp.com/vault/1.18.3/vault_1.18.3_linux_amd64.zip

# 2. 압축 해제
unzip vault_1.18.3_linux_amd64.zip

# 3. 실행 권한 및 이동
sudo chmod +x vault
sudo mv vault /usr/local/bin/

# 4. 확인
vault version
```

## Windows 설치

### 방법 1: Chocolatey

```powershell
# Chocolatey로 설치
choco install vault

# 확인
vault version
```

### 방법 2: 바이너리 다운로드

1. [Vault 다운로드 페이지](https://www.vaultproject.io/downloads) 방문
2. Windows 64-bit 버전 다운로드
3. ZIP 파일 압축 해제
4. `vault.exe`를 `C:\Windows\System32\` 또는 원하는 경로에 복사
5. 환경 변수 PATH에 추가

```powershell
# PowerShell에서 확인
vault version
```

## Docker로 실행 (설치 없이)

```bash
# 일회성 실행
docker run --rm -it hashicorp/vault:latest version

# 별칭 생성 (bash/zsh)
alias vault='docker run --rm -it -e VAULT_ADDR=$VAULT_ADDR -e VAULT_TOKEN=$VAULT_TOKEN hashicorp/vault:latest'

# 사용
vault status
```

## 설치 후 설정

### 1. 자동 완성 활성화

**Bash:**
```bash
vault -autocomplete-install
source ~/.bashrc
```

**Zsh:**
```bash
vault -autocomplete-install
source ~/.zshrc
```

**Fish:**
```bash
vault -autocomplete-install
source ~/.config/fish/config.fish
```

### 2. 환경 변수 설정

**임시 설정:**
```bash
export VAULT_ADDR=https://openbao.cocdev.co.kr
export VAULT_TOKEN=your-token-here
```

**영구 설정:**

**Bash** (`~/.bashrc` 또는 `~/.bash_profile`):
```bash
echo 'export VAULT_ADDR=https://openbao.cocdev.co.kr' >> ~/.bashrc
source ~/.bashrc
```

**Zsh** (`~/.zshrc`):
```bash
echo 'export VAULT_ADDR=https://openbao.cocdev.co.kr' >> ~/.zshrc
source ~/.zshrc
```

**Fish** (`~/.config/fish/config.fish`):
```fish
set -Ux VAULT_ADDR https://openbao.cocdev.co.kr
```

**Windows PowerShell:**
```powershell
[System.Environment]::SetEnvironmentVariable('VAULT_ADDR', 'https://openbao.cocdev.co.kr', [System.EnvironmentVariableTarget]::User)
```

### 3. 토큰 저장 위치

Vault CLI는 로그인 시 토큰을 다음 위치에 저장합니다:

- **macOS/Linux**: `~/.vault-token`
- **Windows**: `%USERPROFILE%\.vault-token`

## OpenBao 연결 테스트

```bash
# 1. 서버 주소 설정
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 2. 연결 확인 (인증 불필요)
vault status

# 3. 로그인
vault login

# 또는 특정 인증 방법 사용
vault login -method=userpass username=admin

# 4. 시크릿 읽기
vault kv get secret/server/staging

# 5. 시크릿 쓰기
vault kv put secret/test/example key=value

# 6. 시크릿 목록
vault kv list secret/server/
```

## 버전 확인 및 업데이트

### 현재 버전 확인
```bash
vault version
```

### 최신 버전 확인
```bash
# 공식 웹사이트
https://www.vaultproject.io/downloads

# 또는 GitHub
https://github.com/hashicorp/vault/releases
```

### 업데이트

**macOS (Homebrew):**
```bash
brew upgrade hashicorp/tap/vault
```

**Ubuntu/Debian:**
```bash
sudo apt update && sudo apt upgrade vault
```

**바이너리 설치:**
새 버전을 다운로드하여 기존 파일 덮어쓰기

## 트러블슈팅

### "command not found: vault"

**원인**: PATH 환경 변수에 vault가 없음

**해결:**
```bash
# vault 위치 확인
which vault

# PATH에 추가
export PATH=$PATH:/usr/local/bin

# 영구 적용
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

### "certificate verify failed"

**원인**: HTTPS 인증서 검증 실패

**해결:**
```bash
# 방법 1: 인증서 확인 스킵 (테스트 환경만)
export VAULT_SKIP_VERIFY=true

# 방법 2: CA 인증서 지정
export VAULT_CACERT=/path/to/ca.crt

# 방법 3: 시스템 CA 업데이트
# macOS
brew install ca-certificates

# Ubuntu
sudo apt install ca-certificates
sudo update-ca-certificates
```

### "connection refused"

**원인**: 서버 연결 불가

**해결:**
```bash
# 1. VAULT_ADDR 확인
echo $VAULT_ADDR

# 2. 서버 상태 확인
curl -k https://openbao.cocdev.co.kr/v1/sys/health

# 3. kubectl port-forward 사용
kubectl port-forward -n openbao svc/openbao 8200:8200
export VAULT_ADDR=http://localhost:8200
```

### "permission denied"

**원인**: 토큰 권한 부족

**해결:**
```bash
# 1. 현재 토큰 확인
vault token lookup

# 2. 정책 확인
vault token lookup | grep policies

# 3. 필요 시 재로그인
vault login
```

## 유용한 별칭 (Alias)

`~/.bashrc` 또는 `~/.zshrc`에 추가:

```bash
# OpenBao 프로덕션
alias vop='export VAULT_ADDR=https://openbao.cocdev.co.kr'

# 로컬 port-forward
alias vlocal='export VAULT_ADDR=http://localhost:8200'

# 빠른 상태 확인
alias vs='vault status'

# 빠른 로그인
alias vl='vault login'

# 시크릿 읽기 (예: vget secret/server/staging)
alias vget='vault kv get'

# 시크릿 목록 (예: vlist secret/server/)
alias vlist='vault kv list'

# 토큰 정보
alias vtoken='vault token lookup'
```

사용 예:
```bash
vop              # 프로덕션 서버 설정
vs               # 상태 확인
vl               # 로그인
vget secret/server/staging  # 시크릿 읽기
```

## 추가 도구

### jq (JSON 파싱)

```bash
# macOS
brew install jq

# Ubuntu/Debian
sudo apt install jq

# 사용 예
vault kv get -format=json secret/server/staging | jq '.data.data'
```

### yq (YAML 파싱)

```bash
# macOS
brew install yq

# Ubuntu/Debian
sudo add-apt-repository ppa:rmescandon/yq
sudo apt update && sudo apt install yq
```

## 참고 자료

- [Vault CLI Documentation](https://www.vaultproject.io/docs/commands)
- [OpenBao Documentation](https://openbao.org/docs/)
- [Vault Installation Guide](https://www.vaultproject.io/docs/install)
- [Vault Commands](https://www.vaultproject.io/docs/commands)
