
## Concourse Pipeline (configure-stage) Task 실행 시 Credhub 확인

- 해당 문서는 Health Watch Tile의 Concourse Staged-config를 통해 출력 된 결과물을 바탕으로 Configure-stage를 실행하여 다시 Ops Manager에 반영 할 때 사용 되는 Bosh Health Check의 값을 구성하는 예시입니다.

- Concourse Staged-config 실행 시 `SUBSTITUTE_CREDENTIALS_WITH_PLACEHOLDERS: true` 옵션 또는 아무것도 명시 하지않게 되면 github commit 결과물에 `((properties_boshtasks_enable_bosh_taskcheck_password.secret))` 와 같게 출력이 되는데 이를 치환하지 않게 되면 Pipeline 실행 시 에러가 발생하게 됩니다.

- 지난 Platform Install/Upgrade 시 `SUBSTITUTE_CREDENTIALS_WITH_PLACEHOLDERS: true` 값을 사용하여 모든 민감한 정보를 출력하여 사용 했습니다. 하지만 해당 예시로 PAS의 속성 값을 Credhub를 통해 안전하게 보관 할 수 있을 것으로 추측 됩니다.

## 1. Health Watch Credhub 정보

### 1.1. Configure-stage Task 실행 시 Error Log
- Configure Task를 실행 시 (()) 값이 치환 되지 않으면 아래와 같은 에러가 발생합니다.

```
Could not find values for:
files/config/p-healthwatch-x.x.x.yml:28 ((properties_boshtasks_enable_bosh_taskcheck_password.secret))
```

#### 1.1.1. Health Watch Credhub 정보 확인
- 위 에러의 값 `properties_boshtasks_enable_bosh_taskcheck_password.secret`은 Healthcheck가 초기 설치 될 경우 생성한 Credhub 종류입니다. 해당 Credhub의 정보는 Bosh Director에 저장되어 있고 정보를 확인하기 위해서는 Bosh Director Credhub에 Targeting 해야 합니다.

#### 1.1.2. Bosh Director Credhub Targeting

```
$ credhub api https://{BOSH_CREDHUB_IP}:8844  --skip-tls-validation
$ credhub login --client-name=ops_manager --client-secret=SSk7cv_XWf-YN3lYvrrqtQGHeC-BXWjN

접근 정보
BOSH_CREDHUB_IP: Bosh Director의 IP
client-name: Ops Manager Credhub 접근 권한이 있는 client name
client-secret: Ops Manager Credhub 접근 권한이 있는 client secret
```

#### 1.1.3. `properties_boshtasks_enable_bosh_taskcheck_password` 값 확인

```
# key의 주요 키워드로 검색합니다.
$ credhub find | grep bosh_taskcheck_password 
- name: /opsmgr/p-healthwatch/boshtasks/enable/bosh_taskcheck_password

# 정보 확인, 이때 type을 확인 합니다.healthwatch의 경우 json type으로 생성 되어 있습니다.
$ credhub get -n /opsmgr/p-healthwatch/boshtasks/enable/bosh_taskcheck_password
id: 8058856a-2555-457f-ad37-7e199b291d5c
name: /opsmgr/p-healthwatch/boshtasks/enable/bosh_taskcheck_password
type: json
value:
  value: Z8ykBt1_RFT2PQ0BwQAFMktPFr0WTbIY
version_created_at: "2020-02-13T01:53:36Z"
```


#### 1.1.4. Concourse Credhub에 `properties_boshtasks_enable_bosh_taskcheck_password`  값 설정
- Pipeline 실행 시 바로 적용 되도록 Concourse Credhub에 해당 값을 적용 합니다. 이때 healthwatch의 예시는 json type입니다.

```
$ credhub api https://concourse.{DOMAIN}:8844   --skip-tls-validation
$ credhub login --client-name=concourse_to_credhub  --client-secret=8kwj6fyryrqkcwy0m1gz

# 이때 secret의 값은 properties_boshtasks_enable_bosh_taskcheck_password.secret 중 .다음인 secret으로 적용합니다.
$ credhub set -t json -n /concourse/admin/properties_boshtasks_enable_bosh_taskcheck_password  -v '{"secret":"fq3dsa78545&$1dsvcx1"}'
```

#### 1.1.5. Pipeline 수정
- Configure-stage Task의 configuration -> configuration-interpolated로 값을 변경하여 Credhub의 값과 Merge 되도록 설정 합니다.

## 2. ETC
### 2.1. 기타 Tile을 예시로 PAS의 인증서 키워드

```
$ credhub find | grep networking_poe_ssl_certs
- name: /opsmgr/cf-bab70e2b133a26a6eacc/networking_poe_ssl_certs/0/certificate
$ credhub find | grep service_provider_key_credentials
- name: /opsmgr/cf-bab70e2b133a26a6eacc/uaa/service_provider_key_credentials

$ credhub get -n /opsmgr/cf-bab70e2b133a26a6eacc/uaa/service_provider_key_credentials


id: 016c98fb-13fe-4ab3-8a55-eb3b5763ba2c
name: /opsmgr/cf-bab70e2b133a26a6eacc/uaa/service_provider_key_credentials
type: json
value:
  cert_and_private_key_pems: |
    -----BEGIN CERTIFICATE-----
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    -----END CERTIFICATE-----

    -----BEGIN RSA PRIVATE KEY-----
    -----END RSA PRIVATE KEY-----
  cert_pem: |
    -----BEGIN CERTIFICATE-----
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    -----END CERTIFICATE-----
  private_key_pem: |
    -----BEGIN RSA PRIVATE KEY-----
    -----END RSA PRIVATE KEY-----
  public_key_pem: |
    -----BEGIN PUBLIC KEY-----
    -----END PUBLIC KEY-----
version_created_at: "2020-02-13T03:33:17Z"
```
- 해당 정보를 통해 JSON Type으로 Concourse Credhub에 인증서 정보를 적용 시키면 github에 민감한 정보를 (())으로 가린채 Apply Change를 실행 할 수 있습니다.
