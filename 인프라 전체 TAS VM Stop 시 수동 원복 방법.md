# 인프라 전체 TAS VM Stop 시 수동 원복 방법
- Pivotal Application Service (TAS)에서 인프라 상에 VM 들을 강제적으로 전원을 STOP 종료 후 다시 복원 시키는 방법

## 1. 사전 준비

### 1.1. Bosh Enable VM Resurrector Plugin  Off
- PCF 상의 VM들을 다시 원복 시키기 위해 자동 복구 기능인 Resurrector 기능이 On 되어 있으면 Bosh Director가 자동으로 복구하여 많은 Task Queue가 쌓이게 되며, 순서가 지정되어 있지 않아 복구 시 많은 시간이 걸릴 수 있다.
- Ops Manager Bosh Director Config에서 Bosh Enable VM Resurrector Plugin Off 시킨다.
- 만약 Director Bosh Enable VM Resurrector Plugin을 On 시킨 상태로 Bosh를 실행 시켰을 때에는 Director VM에 접속하여 별도의 Console로 Task를 Cancel 하는 작업이 필요하다.

### 1.2. PCF Back UP (고려사항)
- 절대 삭제 되지 말아야하는 Data가 존재 할 경우 bbr cli를 이용하여 PCF backup 한다.

#### 1.2.1 bbr CLI Install

```
# jumpbox에 bbr cli를 설치 한다.
# bbr cli 설치 주소 https://github.com/cloudfoundry-incubator/bosh-backup-and-restore/releases
$ wget https://github.com/cloudfoundry-incubator/bosh-backup-and-restore/releases/download/v1.5.1/bbr-1.5.1-linux-amd64

$ chmod +x bbr-1.5.*

$ mv bbr-1.5.* /usr/local/bin/
```

#### 1.2.2. bbr backup example (bosh)

```
# director backup을 하기전 backup disk 용량을 최소화와 빠른 실행을 위해 director를 clean up 한다.

# bosh director가 사용하지 않는 stemcell/release/metadata/disk를 clean 한다.
$ bosh -e ${alias-env} clean-up --all

# bosh director의 backup-restore를 clean 한다.
$ bbr director \
--private-key-path ${BOSH_PRIVATE_KEY_FILE} \
--username bbr \
--host ${BOSH_DIRECTOR_IP} \
backup-cleanup

# bosh director가 backup이 가능한지 pre-backup-check을 실행한다.
$ bbr director --host ${BOSH_DIRECTOR_IP} \
--username bbr  \
--private-key-path ${BOSH_PRIVATE_KEY_FILE} \
pre-backup-check

# bosh director backup을 실행 한다.
$ bbr director \
--private-key-path ${BOSH_PRIVATE_KEY_FILE} \
--username bbr \
--host ${BOSH_DIRECTOR_IP}\
--artifact-path ./generated-backup \
backup

# bosh director의 backup-restore를 다시 clean 한다.
$ bbr director \
--private-key-path ${BOSH_PRIVATE_KEY_FILE} \
--username bbr \
--host ${BOSH_DIRECTOR_IP} \
backup-cleanup

# bosh director backup이 완료되면 artifact-path 디렉토리에 아래와 같은 파일 구조가 생기는 것을 확인 한다.

total 7219824
drwxr-xr-x 2 ubuntu ubuntu       4096 Jun  5 11:12 ./
drwxrwxr-x 3 ubuntu ubuntu       4096 May 30 15:56 ../
-rw-rw-r-- 1 ubuntu ubuntu      30720 May 30 15:49 bosh-0-bbr-credhubdb.tar
-rw-rw-r-- 1 ubuntu ubuntu 7391959040 May 30 15:49 bosh-0-blobstore.tar
-rw-rw-r-- 1 ubuntu ubuntu    1054720 May 30 15:49 bosh-0-director.tar
-rw-rw-r-- 1 ubuntu ubuntu      35723 May 30 15:51 metadata
```
#### 1.2.3.  bbr backup example (pcf)

```
# bosh pivotal cloud foundry의 backup-restore를 clean 한다.
$ bbr deployment \
--target "${BOSH_ENVIRONMENT}" \
--username ${BOSH_CLIENT} \
--TASsword ${BOSH_CLIENT_SECRET} \
--deployment "${DEPLOYMENT_NAME}" \
--ca-cert "${BOSH_CA_CERT}" \
backup-cleanup

# bosh pivotal cloud foundry를 backup이 가능한지 pre-backup-check을 실행한다.
$ bbr deployment \
--target "${BOSH_ENVIRONMENT}" \
--username ${BOSH_CLIENT} \
--TASsword ${BOSH_CLIENT_SECRET} \
--deployment "${DEPLOYMENT_NAME}" \
--ca-cert "${BOSH_CA_CERT}" \
pre-backup-check

# bosh pivotal cloud foundry backup을 실행 한다.
$ bbr deployment \
--target "${BOSH_ENVIRONMENT}" \
--username ${BOSH_CLIENT} \
--TASsword ${BOSH_CLIENT_SECRET} \
--deployment "${DEPLOYMENT_NAME}" \
--ca-cert "${BOSH_CA_CERT}" \
--artifact-path ./generated-backup \
backup

# bosh pivotal cloud foundry의 backup-restore를 다시 clean 한다.
$ bbr deployment \
--target "${BOSH_ENVIRONMENT}" \
--username ${BOSH_CLIENT \
--TASsword ${BOSH_CLIENT_SECRET \
--deployment "${DEPLOYMENT_NAME" \
--ca-cert "${BOSH_CA_CERT}" \
backup-cleanup

# bosh pivotal cloud foundry backup이 완료되면 artifact-path 디렉토리에 아래와 같은 파일 구조가 생기는 것을 확인 한다.

total 76920436
drwx------ 2 ubuntu ubuntu        4096 May 24 19:45 ./
drwxrwxr-x 5 ubuntu ubuntu        4096 May 29 15:17 ../
-rw-r--r-- 1 ubuntu ubuntu       10240 May 24 19:45 backup_restore-0-azure-blobstore-backup-restorer.tar
-rw-r--r-- 1 ubuntu ubuntu       20480 May 24 19:45 backup_restore-0-backup-restore-notifications.tar
-rw-r--r-- 1 ubuntu ubuntu       20480 May 24 19:31 backup_restore-0-backup-restore-pcf-autoscaling.tar
-rw-r--r-- 1 ubuntu ubuntu     1249280 May 24 19:45 backup_restore-0-bbr-cfnetworkingdb.tar
-rw-r--r-- 1 ubuntu ubuntu   204410880 May 24 19:45 backup_restore-0-bbr-cloudcontrollerdb.tar
-rw-r--r-- 1 ubuntu ubuntu       30720 May 24 19:31 backup_restore-0-bbr-credhubdb.tar
-rw-r--r-- 1 ubuntu ubuntu       10240 May 24 19:45 backup_restore-0-bbr-routingdb.tar
-rw-r--r-- 1 ubuntu ubuntu      317440 May 24 19:45 backup_restore-0-bbr-uaadb.tar
-rw-r--r-- 1 ubuntu ubuntu   105984000 May 24 19:45 backup_restore-0-bbr-usage-servicedb.tar
-rw-r--r-- 1 ubuntu ubuntu       10240 May 24 19:45 backup_restore-0-nfsbroker-bbr.tar
-rw-r--r-- 1 ubuntu ubuntu       10240 May 24 19:31 backup_restore-0-s3-unversioned-blobstore-backup-restorer.tar
-rw-r--r-- 1 ubuntu ubuntu       10240 May 24 19:45 backup_restore-0-s3-versioned-blobstore-backup-restorer.tar
-rw-r--r-- 1 ubuntu ubuntu      164153 May 24 19:31 metadata
-rw-r--r-- 1 ubuntu ubuntu       10240 May 24 19:31 nfs_server-0-blobstore-backup.tar
-rw-r--r-- 1 ubuntu ubuntu 78454231040 May 24 19:45 nfs_server-0-blobstore.tar
```

### 1.3. TAS Mysql Instance 조정 (1개일 경우 생략)
- TAS 사용 Mysql의 수를 1개로 조정하여 TAS 만을 Apply Change 한다.

## 2. Control VM(Bosh, Ops Manager) Start
- PCF 상의 모든 VM의 Lifecycle을 관리하는 Bosh VM과, 해당 Bosh가 Targeting 되어 있는 Ops Manager VM, JumpBox의 전원을 On 시킨다.

## 3. Bosh CLI를 통한 복구

### 3.1. Bosh CLI를 통해 PCF 상의 전체 VM을 1차적으로 원복 시킨다.

```
$ bosh -e {ailas-env} cck reboot_vm
```

### 3.2. Bosh CCK를 통해 VM을 Start 했을 때 상태 Error
- Timed out sending 'get_state' 에러 메세지가 포함 됬을 경우
- $ bosh -e {ailas-env} vms 중 특정 VM 상태가 알 수 없음 상태가 있을 경우
```
# 해당 명령어로 통하여 이전 Success 상태로 원복한다.
$ bosh -e {alias-env} -d cf-xxx recreate --fix {job_name}/{job_uuid}
```
