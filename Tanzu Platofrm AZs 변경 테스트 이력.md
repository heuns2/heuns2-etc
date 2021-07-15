# Tanzu Platofrm AZs 변경 테스트 이력

- Tanzu Platform의 모든 Tile은 IaaS의 설정한 Availability Zone에 기반하여 VM이 배포 됩니다. VM이 한번 배포 되면 AZs 설정 편집 UI는 Disable 됩니다.
- Availability Zone 설정은 Bosh Director 설정에 존재하고 있습니다.
- Openstack Zone을 추가하고 하였으며 해당 부분에서 Tanzu Platform에서 신규 AZs를 추가하고 기존 AZs에 있는 모든 VM을 옮기려고하는 요구 조건이 발생하여 테스트를 진행 한 이력입니다.
- 테스트의 결과는 아래왜 같습니다.

` Single Node를 포함하여 모든 VM의 AZs를 옮길 경우 Nfs Server, Database의 Persistence 영역이 유실되어 장애가 초래 할 수 있습니다. 만약에 해당 작업을 고려하시면 AZs를 Balance other jobs in에만 추가해서 사용하여 기존 VM들의 분리하는 방안이 있을 것 같습니다, 그리고 사전에 TAS Backup 본이 반드시 필요합니다.`

## 1. Tile의 AZs 설정 UI를 Admin 권한으로 Disable -> Enable 형태로 변경

- AZs를 설정하여 초기 1회 배포 시 해당 칸은 Read Only 상태로 변경되어 편집 불가능 합니다. 하지만 Lock을 해제 하여 고급 옵션을 활성화하여 편집이 가능 한 상태로 변경합니다.



```
# UAA Targeting
$ uaac target https://<OPSMANAGER_IP>/uaa --skip-ssl-validation
# UAA 인증 획득
$ uaac token owner get
Client ID: opsman
Client secret:<공백 ENTER>
User name:  <OPSMANAGER UI ID>
Password:  <OPSMANAGER UI PWD>
# Unlock 수행
$ uaac curl -k https://{OPSMANAGER_IP}/api/v0/staged/infrastructure/locked -X PUT --data '{"locked" : "false"}' -H "Content-Type: application/json"
```

## 2. 테스트 이력

### 2.1. 기존 AZs 설정은 그대로 두고 신규 Azs만 Balance other jobs in에 추가 할 경우

- Apply Change 후 TAS 형상

![azs-1][azs-1]

[azs-1]:./images/azs-image-1.PNG

- Apply Change 후 TAS 형상

![azs-2][azs-2]

[azs-2]:./images/azs-image-2.PNG

`생성한 Json 형태의 Service, App, Route 등 기존 서비스가 정상적으로 동작하는지 확인 완료`

### 2.2. 신규 AZs 만 사용하도록 Place singleton jobs in, Balance other jobs in를 모두 한쪽으로 지정 할 경우 (기존 AZs를 싹 비울 경우)

- 장애 발생

![azs-3][azs-3]

[azs-3]:./images/azs-image-3.PNG

`App의 Request, 모든 Database 상의 형상 확인 불가`
