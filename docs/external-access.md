# OpenBao 외부 접근 가이드

## 현재 설정 (프로덕션 환경)

OpenBao는 다음과 같이 설정되어 있습니다:

- **도메인**: `https://openbao.cocdev.co.kr`
- **인증서**: Let's Encrypt (자동 갱신)
- **Ingress**: nginx-ingress-controller
- **TLS Secret**: `openbao-tls-cert`

## 1. HTTPS를 통한 외부 접근 (권장) ✅

### Vault CLI 설정

```bash
# 환경 변수 설정
export VAULT_ADDR=https://openbao.cocdev.co.kr

# 로그인
vault login

# 상태 확인
vault status
```

### Bruno/Postman에서 사용

**Base URL**: `https://openbao.cocdev.co.kr`

**예시 요청**:
```http
POST https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin
Content-Type: application/json

{
  "password": "your-password"
}
```

**응답에서 토큰 추출**:
```json
{
  "auth": {
    "client_token": "hvs.CAESIxxxxxx"
  }
}
```

**토큰으로 시크릿 읽기**:
```http
GET https://openbao.cocdev.co.kr/v1/secret/data/server/staging
X-Vault-Token: hvs.CAESIxxxxxx
```

### curl 사용

```bash
# 헬스 체크
curl https://openbao.cocdev.co.kr/v1/sys/health

# 로그인
TOKEN=$(curl -s -X POST \
  https://openbao.cocdev.co.kr/v1/auth/userpass/login/admin \
  -d '{"password":"your-password"}' | jq -r '.auth.client_token')

# 시크릿 읽기
curl -H "X-Vault-Token: $TOKEN" \
  https://openbao.cocdev.co.kr/v1/secret/data/server/staging
```

## 2. kubectl port-forward (개발/디버깅)

로컬 개발 환경에서 사용:

```bash
# 포트 포워딩 시작
kubectl port-forward -n openbao svc/openbao 8200:8200

# 다른 터미널에서
export VAULT_ADDR=http://localhost:8200
vault login

# 또는 특정 포트 사용
kubectl port-forward -n openbao svc/openbao 9200:8200
export VAULT_ADDR=http://localhost:9200
```

**장점**:
- 빠른 테스트
- 인증서 걱정 없음
- 방화벽 우회

**단점**:
- 터미널 세션이 끊기면 연결 종료
- 한 번에 한 사용자만 가능

## 3. NodePort 설정 (테스트 환경)

### 서비스를 NodePort로 변경

```bash
kubectl patch svc openbao -n openbao -p '{"spec":{"type":"NodePort"}}'

# 할당된 포트 확인
kubectl get svc openbao -n openbao
```

### 접근 방법

```bash
# 노드 IP 확인
kubectl get nodes -o wide

# 노드 IP와 NodePort로 접근
export VAULT_ADDR=http://<NODE_IP>:<NODE_PORT>
vault login
```

**주의사항**:
- 프로덕션에서는 권장하지 않음
- 보안 위험 (HTTPS 없음)
- 포트가 랜덤으로 할당됨 (30000-32767)

## 4. LoadBalancer 설정 (클라우드 환경)

클라우드 환경(AWS, GCP, Azure)에서만 사용 가능:

```bash
kubectl patch svc openbao -n openbao -p '{"spec":{"type":"LoadBalancer"}}'

# 외부 IP 확인 (할당까지 시간 소요)
kubectl get svc openbao -n openbao -w
```

**비용 발생 주의!**

## 5. SSH 터널 (보안 접근)

점프 서버를 통한 접근:

```bash
# SSH 터널 생성
ssh -L 8200:openbao.cocdev.co.kr:443 user@bastion-server

# 로컬에서 접근
export VAULT_ADDR=http://localhost:8200
vault login
```

## Bruno 컬렉션 설정 예시

### 환경 변수 설정

**environments/production.bru**:
```javascript
vars {
  base_url: https://openbao.cocdev.co.kr
  token: ""
}
```

**environments/local.bru**:
```javascript
vars {
  base_url: http://localhost:8200
  token: ""
}
```

### 로그인 요청

**OpenBao/login.bru**:
```javascript
meta {
  name: Login
  type: http
  seq: 1
}

post {
  url: {{base_url}}/v1/auth/userpass/login/{{username}}
  body: json
}

body:json {
  {
    "password": "{{password}}"
  }
}

script:post-response {
  if (res.status === 200) {
    bru.setEnvVar('token', res.body.auth.client_token);
    console.log('✅ 로그인 성공');
    console.log('Token:', res.body.auth.client_token);
  } else {
    console.log('❌ 로그인 실패');
  }
}
```

### 시크릿 읽기

**OpenBao/read-secret.bru**:
```javascript
meta {
  name: Read Secret
  type: http
  seq: 2
}

get {
  url: {{base_url}}/v1/secret/data/server/staging
}

headers {
  X-Vault-Token: {{token}}
}

script:post-response {
  if (res.status === 200) {
    console.log('✅ 시크릿 읽기 성공');
    console.log(JSON.stringify(res.body.data.data, null, 2));
  }
}
```

### 시크릿 생성/수정

**OpenBao/write-secret.bru**:
```javascript
meta {
  name: Write Secret
  type: http
  seq: 3
}

post {
  url: {{base_url}}/v1/secret/data/server/staging
  body: json
}

headers {
  X-Vault-Token: {{token}}
}

body:json {
  {
    "data": {
      "APP_PORT": "3000",
      "APP_NAME": "plate-api",
      "NODE_ENV": "staging"
    }
  }
}

script:post-response {
  if (res.status === 200 || res.status === 204) {
    console.log('✅ 시크릿 저장 성공');
  }
}
```

## 보안 권장사항

### 1. HTTPS 사용 (현재 설정 유지)
✅ Let's Encrypt 인증서 사용 중
✅ 자동 갱신 설정됨

### 2. 인증서 확인

```bash
# 인증서 상태 확인
kubectl get certificate -n openbao

# cert-manager 로그 확인
kubectl logs -n cert-manager deployment/cert-manager -f
```

### 3. TLS 버전 및 암호화 강화

Ingress에 annotation 추가:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

### 4. IP 화이트리스트 (선택사항)

특정 IP만 접근 허용:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "1.2.3.4/32,5.6.7.8/32"
```

### 5. Rate Limiting

DDoS 방지:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
```

## 트러블슈팅

### 인증서 오류

```bash
# 인증서 상태 확인
kubectl describe certificate openbao-tls-cert -n openbao

# cert-manager 재시작
kubectl rollout restart deployment cert-manager -n cert-manager

# 인증서 강제 갱신
kubectl delete certificate openbao-tls-cert -n openbao
# (자동으로 재생성됨)
```

### DNS 문제

```bash
# DNS 확인
nslookup openbao.cocdev.co.kr

# hosts 파일 임시 설정 (로컬 테스트)
# /etc/hosts에 추가:
192.168.0.20 openbao.cocdev.co.kr
```

### 연결 타임아웃

```bash
# Ingress 로그 확인
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# OpenBao 로그 확인
kubectl logs -n openbao -l app.kubernetes.io/name=openbao -f

# 서비스 엔드포인트 확인
kubectl get endpoints openbao -n openbao
```

## 참고 자료

- [OpenBao Documentation](https://openbao.org/docs/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [cert-manager](https://cert-manager.io/docs/)
- [nginx-ingress](https://kubernetes.github.io/ingress-nginx/)
