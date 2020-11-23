
# Gemfire Region Persistence Disk 영역
- Gemfire Region 생성 시 type을 persistence, eviction-action을 disk_overflow로 생성 한 Region에 대해서는 Bosh Agent로 설치 된 VM의  /var/vcap/store/gemfire-server 영역에 자동으로 DEFAULT Disk가 설정되고 put, delete 또는 eviction시 region의 data가 쌓이게 됩니다.

```
$ list disk-store
                  Member Name                    |                                            Member Id                                            | Disk Store Name | Disk Store ID
------------------------------------------------ | ----------------------------------------------------------------------------------------------- | --------------- | ------------------------------------
cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx | c410xxxx1bb1-xxx-4da3-8a03-xxxx(cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx:12)<v70.. | DEFAULT         | c9e6b3dd-ee2d-4443-b901-f55c9002d765

$ ls -al /var/vcap/store/gemfire-server
-rw-r--r-- 1 vcap                 vcap                 966367641 Sep  2 04:33 BACKUPDEFAULT_13.crf
-rw-r--r-- 1 vcap                 vcap                 107374183 Jul 23 05:09 BACKUPDEFAULT_13.drf
-rw-r--r-- 1 vcap                 vcap                  39059641 Jul 23 05:09 BACKUPDEFAULT_6.crf
-rw-r--r-- 1 vcap                 vcap                       262 Jul 23 05:09 BACKUPDEFAULT_6.drf
-rw-r--r-- 1 vcap                 vcap                    324305 Jun 25 16:24 BACKUPDEFAULT_6.krf
-rw-r--r-- 1 vcap                 vcap                   5242880 Sep  2 04:33 BACKUPDEFAULT.if
```

- File의 의미는 아래와 같습니다.
	- *.crf: region에 create, update 한 data가 쌓이는 파일
	- *.drf: region에 remove한 data가 쌓이는 파일
	- *.if: disk store의 meta data 정보
	- *.krf: .crf 파일의 offset 정보 - 시작시 성능을 향상시키는 데 사용
	- 파일이 여러개 쌓이는 이유는 파일의 사이즈로 rotate

## 1.  Persistence Disk 용량 삭제 방안

### 1.1. Region  재생성
- Region을 destroy -> create

```
$ destroy region --name=/leedh
$ create region --name=/leedh --type=PARTITION_PERSISTENT_OVERFLOW --entry-idle-time-expiration=86000 --entry-idle-time-expiration-action=DESTROY --enable-statistics=true --entry-time-to-live-expiration=86000 --entry-time-to-live-expiration-action=DESTROY 
```

### 1.2. gemfire process 재기동
- directory의 File 강제 삭제 후 gemfire process 재실행 (부분적으로 locator를 stop 시키고 해야 할 수 있습니다.)

```
$ cd /var/vcap/store/gemfire-server/
$ rm -rf *
$ monit restart gemfire-server
```

### 1.3. 확인 방안
- 작업 완료 후 정상적으로 모든 Disk와 Member가 붙어있는지 확인

```
$ list members
Member Count : 7

                      Name                       | Id
------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------
locator-c410xxxx1bb1-xxx-4da3-8a03-xxxx     | c410xxxx1bb1-xxx-4da3-8a03-xxxx(locator-c410xxxx1bb1-xxx-4da3-8a03-xxxx:6:locator)<ec><v48>:56152 [Coordinator]
locator-5bdaba9c-55a9-4b8c-a886-90ee69b89288     | 5bdaba9c-55a9-4b8c-a886-90ee69b89288.locator.service-network.service-instance-c410xxxx1bb1-xxx-4da3-8a03-xxxx.bosh(locator-5bdaba9c-55a9-4b8c-a88..
locator-2258d666-b97e-4a79-bb0e-976c2f84afac     | 2258d666-b97e-4a79-bb0e-976c2f84afac.locator.service-network.service-instance-c410xxxx1bb1-xxx-4da3-8a03-xxxx.bosh(locator-2258d666-b97e-4a79-bb0..
cacheserver-ecee184d-8282-4e99-8f08-d2ddd787958f | ecee184d-8282-4e99-8f08-d2ddd787958f(cacheserver-ecee184d-8282-4e99-8f08-d2ddd787958f:8)<v69>:56152
cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx | c410xxxx1bb1-xxx-4da3-8a03-xxxx(cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx:12)<v70>:56152
cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx | c410xxxx1bb1-xxx-4da3-8a03-xxxx(cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx:8)<v71>:56152
cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx | c410xxxx1bb1-xxx-4da3-8a03-xxxx(cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx:8)<v65>:56152
```
