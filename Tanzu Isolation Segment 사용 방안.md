# Tanzu Isolation Segment 사용 방안

- Tanzu Isolation Segment는 Tanzu Application Service와는 별개로 Network, Container Runtime을 분리 할 수 있습니다, 특히 Network 분리를 통하여 몸통과는 전혀 다른 Network 규칙을 생성 할 수 있으며 Traffic을 컨트롤 할 수 있습니다.
- Tanzu Application Service의 BBS, UAA, Database, NFS Server는 함께 사용 됩니다.

## 1. Tanzu Isolation Segment 설치 진행 시 특이 사항

- Go Router를 분리하고, 별도의 Apps Domain을 사용하기 위해서는 아래의 설정에서 해당 Apps Domain으로 인증서를 생성해야 합니다.

![isg-1][isg-1]
[isg-1]:./images/isg-image-1.PNG

- Diego Cell을 분리하기 위해서는 아래의 설정이 필요하며, 아래 설정을 통하여 Tile 설치 완료 후 

![isg-2][isg-2]
[isg-2]:./images/isg-image-2.PNG


## 2. Tanzu Isolation Segment 설치 후 설정

```
# 설치 단계에 설정한 leedh-is 명칭의 isolation-segment 생성
$ cf create-isolation-segment leedh-is

# isolation-segment로 분리 할 ORG 생성
$ cf create-org isg-test

# isolation-segment org 활성화
$ cf enable-org-isolation isg-test leedh-is

# isolation-segment org 설정
$ cf set-org-default-isolation-segment isg-test leedh-is

# isolation-segment 확인
$ cf isolation-segments
Getting isolation segments as admin...
OK

name       orgs
shared
leedh-is   isg-test

$ cf t -o isg-test
$ cf create-space test

# isolation-segment에서 사용 할 domain 생성
$ cf create-shared-domain apps.isg.leedh.cloud
$ cf domains
Getting domains in org isg-test as admin...
name                   status   type   details
apps.leedh.cloud       shared
apps.internal          shared          internal
apps.isg.leedh.cloud   shared

# app push시 apps.isg-leedh.cloud domain을 사용
$ cf push test-isg -p test-boot-0.0.1-SNAPSHOT.jar -d apps.isg.leedh.cloud 
```

## 3. Tanzu Isolation Segment 설정 후 확인

- APP에 대한 Request가 정상 수행 되는지 확인

```
$ curl https://test-isg.apps.isg.leedh.cloud -k -i
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   192  100   192    0     0    761      0 --:--:-- --:--:-- --:--:--   768HTTP/1.1 200 OK
Content-Length: 192
Content-Type: text/plain;charset=UTF-8
Date: Thu, 15 Jul 2021 02:37:48 GMT
X-Vcap-Request-Id: 455046ad-9f6b-4f02-7f46-28eab775b0a9

application_name=default</br>application_id=2e3bbbac-a930-4acb-89f8-3906144a9457</br>application_uris=test-isg.apps.isg.leedh.cloud</br>cf_api=https://api.sys.leedh.cloud</br>test=default</br>
```

- App의 형상을 확인하여 App의 Host가 ISG Diego Cell의 IP와 같은지 확인


```
$ cf curl /v2/apps/2e3bbbac-a930-4acb-89f8-3906144a9457/stats
{
   "0": {
      "state": "RUNNING",
      "isolation_segment": "leedh-is",
      "stats": {
         "name": "test-isg",
         "uris": [
            "test-isg.apps.isg-leedh.cloud"
         ],
         "host": "10.0.4.40",
         "port": 61000,
         "uptime": 17,
         "mem_quota": 1073741824,
         "disk_quota": 1073741824,
         "fds_quota": 16384,
         "usage": {
            "time": "2021-07-15T02:08:57+00:00",
            "cpu": 0.018352620600578847,
            "mem": 133463071,
            "disk": 135487488
         }
      }
   }
}

Deployment 'p-isolation-segment-b0b23a14da895c67977e'

Instance                                                  Process State  AZ               IPs        VM CID               VM Type   Active  Stemcell
isolated_diego_cell/0ebe99a3-5096-423c-aae7-8576af341456  running        ap-northeast-1a  10.0.4.40  i-0ed79d5a6f4179236  m5.large  true    bosh-aws-xen-hvm-ubuntu-xenial-go_agent/621.117
isolated_router/4facc945-c276-4bbb-8953-94a9cfba2b8b      running        ap-northeast-1a  10.0.4.11  i-0a7aa718c99598b9d  c5.large  true    bosh-aws-xen-hvm-ubuntu-xenial-go_agent/621.117
```
