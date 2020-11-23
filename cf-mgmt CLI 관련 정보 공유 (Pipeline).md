
## cf-mgmt CLI 관련 정보 공유 (Pipeline)
- Tanizu Application Service의 Admin 정보(USER/ORG/SPACE/SERVICE/ASG)를 관리하기 위한 Management CLI
- Concourse Pipeline을 통해 관리 할 수 있다는 이점이 있으며 Git과 연동하여 Version을 관리 할 수 있다는 이점이 있습니다.
- 가장 큰 장점은 cf-mgmt로 admin이 생성한 정보를 github에 이력관리하여 개발자가 따로 불필요한 정보를 생성 하였을 경우 일괄 삭제 처리 할 수 있습니다.

- cf-mgmt github 주소: https://github.com/pivotalservices/cf-mgmt.git

### 1. cf-mgmt cli 종류

#### 1.1. cf-mgmt-config
- pas Foundation의 정보를 추출 할 떄 사용 되는 CLI

#### 1.2. cf-mgmt
- cf-mgmt를 통해 추출한 Foundation 정보를 바탕으로 정보를 일괄 적용/부분 적용 할 수 있는 기능을 제공하는 CLI

### 2. 사용 방법

#### 2.1. uaac 정보 설정
- cf-mgmt-config CLI를 사용하기 위한 권한을 부여하기 위해 PAS 내부에 Client를 생성합니다.

```
uaac target uaa.<system-domain>
uaac token client get <adminuserid> -s <admin-client-secret>

uaac client add cf-mgmt \
  --name cf-mgmt \
  --secret <cf-mgmt-secret> \
  --authorized_grant_types client_credentials,refresh_token \
  --authorities cloud_controller.admin,scim.read,scim.write,routing.router_groups.read
```

#### 2.1. cf-mgmt-config 사용
- cf-mgmt-config를 사용하여 Foundation의 정보 추출

```
# 아래 명령어를 실행 시 Foundation의 전체 configuration 정보가 추출 됩니다.
$ ./cf-mgmt-linux export-config --config-dir=./cf-mgmt \ 
--system-domain={SYSTEM_DOMAIN} \
--user-id=cf-mgmt  --client-secret=cf-mgmt
```

- 주요 추출 사항 (일부 발췌)

1) PAS의 LDAP 정보
```
enabled: false
ldapHost: ""
ldapPort: 0
use_tls: false
bindDN: ""
userSearchBase: ""
userNameAttribute: ""
userMailAttribute: ""
userObjectClass: ""
groupSearchBase: ""
groupAttribute: ""
groupObjectClass: ""
origin: uaa
insecure_skip_verify: ""
ca_cert: ""
useIDForSAMLUser: false
```

2) PAS의 ORG 정보

```
orgs:
- p-spring-cloud-gateway-service
enable-delete-orgs: true
protected_orgs:
- system
- p-spring-cloud-services
- splunk-nozzle-org
- redis-test-ORG*
- appdynamics-org
- credhub-service-broker-org
- p-dataflow
```

3) PAS의 Service 정보
```
enable-delete-isolation-segments: false
enable-unassign-security-groups: false
running-security-groups:
- default_security_group
staging-security-groups:
- default_security_group
shared-domains:
  apps.internal:
    internal: true
  {APPS_DOMAIN}:
    internal: false
enable-remove-shared-domains: true
metadata-prefix: cf-mgmt.pivotal.io
enable-service-access: true
ignore-legacy-service-access: false
service-access:
- broker: cloudcache-broker
  services:
  - service: p-cloudcache
    all_access_plans:
    - extra-large
    - dev-plan
    - small-footprint
    - extra-small
    - small
    - medium
    - large
- broker: smbbroker
  services:
  - service: smb
    no_access_plans:
    - Existin
```

#### 2.2. cf-mgmt-config 사용
- 해당 CLI는 저장되어 있는 Configuration 정보와 diff 후 일괄 적용/부분 적용하는 것으로 추측됩니다.


- 만약 기존 org 정보를 추가하여 Foundation에 적용 할 경우

```
$ ./cf-mgmt-linux create-orgs --config-dir=./cf-mgmt \ 
--system-domain={SYSTEM_DOMAIN} \
--user-id=cf-mgmt  --client-secret=cf-mgmt
```

- 만약 권한이 없는 User를 일괄 삭제 할 경우

```
$ ./cf-mgmt-linux cleanup-org-users --config-dir=./cf-mgmt \ 
--system-domain={SYSTEM_DOMAIN} \
--user-id=cf-mgmt  --client-secret=cf-mgmt
```
