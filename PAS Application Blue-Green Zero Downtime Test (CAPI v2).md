# PAS Application Blue-Green Zero Downtime Test (CAPI v2)
- PAS 상의 Application의 Downtime과 Risk을 줄이기 위해 Application의 Blue/Green 방법을 사용 한다.
- 준비 사항
	- cf CLI를 실행 할 수 있는 Jump box 환경
	- Test를 위한 Sample Application
- Blue-Green의 주요 정책은 Blue 상태 일 경우 현재 사용 중, Green 상태는 배치 및 테스트의 최종 단계를 거치는 상태인 Application이다.
- 배치 및 테스트가 완료되면 Application으로 통하는 Route를 Blue->Green으로 변경해 실제 Route 요청이 되도록 실행 한다.
- 만약 Green에서 예외가 발생 할 경우 이전 버전인 Blue로 다시 Rollback 한다.
- 만약 변경 한 Green이 정상적으로 동작한다면 Green이 Blue가 되고 Blue가 Green이 되는 Application서로의 상태가 변경 된다.

## 1. 일반적인 Application Blue-Green Application Test
- Test Application은 Blue-Green 양방향으로 변경하여 Application이 Downtime 없이 변경 사항이 최신화 되는지 확인 한다.
 
### 1.1. Step 1: Push an App
- Test용 Blue Application을 특정 Sub Domain을 입력하여 배포 한다.
- Blue Application의 Route는 Downtime 없이 사용되는 Application의 접근 경로이다.

```
$ cf push test-blue -n test-blue
```

### 1.2. Step 2: Update App and Push
- Test용 Green Application에서 변경사항이 있는 특정 Blue Application Sub Domain을 입력하여 배포 한다.

```
$ cf push test-green -n test-green
```

- 다음 단계 테스트 전 실제 Blue Application의 경로 접근 시 Downtime(경로에 대한 404)이 발생 되는지 CURL을 통해 확인 한다.

```
$ while true; do curl https://test-blue.{YOUR_DOMAIN} -k; sleep 1 done
```

### 1.3. Step 3: Map Original Route to Green
- cf map-route 명령을 사용하여 현재 사용하는 Blue Application의 URL 경로 (test-green.{SYSTEM_DOMAIN})를 Green Application에 Mapping 한다.

```
$ cf map-route test-green apps.{YOUR_DOMAIN} -n test-blue
Binding demo-time.example.com to Green... OK
```
- Blue-Green Route 상황을 확인 한다.

```
$ cf apps
test-blue             started           2/2         1G       1G     test-blue.{YOUR_DOMAIN}
test-green            started           1/1         1G       1G     test-blue.{YOUR_DOMAIN}, test-green.{YOUR_DOMAIN}
```

### 1.4. Step 4: Unmap Route to Blue
- Green Application에 문제가 없으면 기존에 남아 있던 Blue Application에 Mapping 된 Route 정보를 해제 한다.

```
$ cf unmap-route test-blue {YOUR_DOMAIN} -n test-blue
Unbinding test-blue.SYSTEM_DOMAIN from blue... OK
```
- 정상적으로 적용 되었다면 Green Application의 Green Route Mapping을 해제 한다.

```
$ cf unmap-route test-green {YOUR_DOMAIN} -n test-green
```

- 기존의 Blue로 통신되던 Route 경로는 변경한 Green Application으로만 Mapping 된다.

### 1.5. Step 5: CURL 실행 결과 확인
- 1.2 Step의 $ while true; do curl https://test-blue.{YOUR_DOMAIN}-k; sleep 1 done 실행 결과 중 404가 출력 되지 않았는지(Downtime이 없는지) 확인 한다.
	- Example: 404 Not Found: Requested route ('test-green.{YOUR_DOMAIN}') does not exist.

- 이제 최종적으로 남은 Green Application이 실제 가동 상태인 Blue Application으로 변동 되었으며 추 후 변경한 Application이 Green Application으로 배포 된다.

## 2. Plug-In 사용 Application Blue-Green Application Test
- Blue/Green deployer plugin for CF는1단계에서 실행 했던 기본 Blue-Green Application Test를 몆 단계 자동화 한 CLI를 제공한다.

- Plug-In 설치 방법: 

```
$ cf add-plugin-repo CF-Community https://plugins.cloudfoundry.org
$ cf install-plugin blue-green-deploy -r CF-Community
```
- Plug-In 설치 확인

```
$ cf blue-green-deploy -h
NAME:
   blue-green-deploy - Zero-downtime deploys with smoke tests

ALIAS:
   bgd

USAGE:
   blue-green-deploy APP_NAME [--smoke-test TEST_SCRIPT] [-f MANIFEST_FILE] [--delete-old-apps]

OPTIONS:
   -delete-old-apps       Delete old app instance(s)
   -f                     Path to manifest
   -smoke-test            The test script to run.
```

### 2.1. Step 1: Blue Application CF Push
- 일반적인 명령어를 사용하여 Blue Application을 Push 한다

```
$ cf push test-blue -n test-blue
``` 

### 2.2. Step 2: blue-green-deploy Plugin CF Push
- blue-green-deploy Plugin을 사용하여 새로운 버전의 Green Application을 Push하고 변경사항이 잘 적용 되었는지 확인 한다.
- blue-green-deploy 명령어를 통해 자동으로 Green Application 배포 -> Green  Router Mapping -> Blue Application, Green Application 이름 변경 -> Blue Router UnMapping 된다.

```
$ cf blue-green-deploy test-blue -f manifest.yml

Removing route test-blue-new.{YOUR_DOMAIN} from app test-blue-new in org  {YORE_ORG} / space test as pcfadmin...
OK
Deleting route test-blue-new.{YOUR_DOMAIN}...
OK
Creating route test-blue.{YOUR_DOMAIN} for org {YORE_ORG}/ space test as pcfadmin...
OK
Route test-blue.{YOUR_DOMAIN} already exists
Adding route test-blue.{YOUR_DOMAIN} to app test-blue-new in org  {YORE_ORG} / space test as pcfadmin...
OK
Renaming app test-blue to test-blue-old in org  {YORE_ORG} / space test as pcfadmin...
OK
Renaming app test-blue-new to test-blue in org  {YORE_ORG} / space test as pcfadmin...
OK
Removing route test-blue.{YOUR_DOMAIN} from app test-blue-old in org  {YORE_ORG} / space test as pcfadmin...
OK
```
- 배포 성공 후 Old Application을 삭제 할 경우 -delete-old-apps 옵션을 사용한다.

- 최종 Application 형상

```
$ cf apps
test-blue             started           1/1         1G       1G     test-blue.{YOUR_DOMAIN}
test-blue-old         started           1/1         1G       1G
```

### 2.3. Step 3: blue-green-deploy Plugin 주의 사항
- Multi Buildpack Manifest 파일을 사용하여 Push 했을 경우 아래 에러 메세지가 출력 한다.

```
$ cf blue-green-deploy test-blue
Could not push new version - Error reading manifest file:
The following manifest fields cannot be used with legacy push: buildpacks
```


- 아래 설정을 변경하여 배포 후 성공

```
# 변경 전
---
applications:
- name: spring-music
  memory: 1G
  buildpacks: 
  - java_buildpack_offline
  path: ./spring-music.jar

# 변경 후
---
applications:
- name: spring-music
  memory: 1G
  buildpack: java_buildpack_offline
  #- java_buildpack_offline
  path: ./spring-music.jar
```

## 3. Plug-In 사용 Application Blue-Green Application Error Test
- 만약 Green Application이 Crash 상태로 된 Application을 배포 할 경우의 동작에 대한 Test를 진행 한다.

### 3.1. 정상 동작 Application 배포
```
$ cf push test-blue -n test-blue
```

### 3.2. Plug-In을 사용하여 Crash Application 배포
```
$ cf blue-green-deploy test-blue -f manifest.yml
Cell 4f5a576b-b08d-4b46-95fb-84a5f92b356a successfully destroyed container for instance 84406ee7-162a-4543-ae8d-81e962a8c6da

0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 crashed
Could not push new version - Error restarting application: Start unsuccessful

TIP: use 'cf logs test-blue-new --recent' for more information
```
- test-blue-new Application 배포에 실패하고 더 이상 진행 되지 않는다.
