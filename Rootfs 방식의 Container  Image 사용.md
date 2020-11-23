# Rootfs 방식의 Container  Image 사용
- rootfs는 linux kernel를 생성 할 때 사용되는 Linux File System 입니다. 많이 봐왔던 /usr, /bin, /etc, /media, /proc 등으로 구성 할 수 있습니다.
- TAS에서의 Rootfs는 Stack으로 표현 됩니다.  Stack은 App이 가동되고 있는 환경의 Base File System입니다.

```
$ cf stacks
Getting stacks in org patch / space m2a as admin...
OK

name            description
cflinuxfs3      Cloud Foundry Linux-based filesystem - Ubuntu Bionic 18.04 LTS
windows2012R2   Microsoft Windows / .Net 64 bit
windows2016     Microsoft Windows 2016
windows         Windows Server
```

- TAS에서 Stack의 사용은 Garden Run 구성의 일부인 grootfs(Docker 및 Open Container Initiative) Image를 처리하는 Process가 App을 재기동 할 때 Stack 또는 Docker Image를 처리 합니다. [grootfs 설명 문서](https://github.com/cloudfoundry/grootfs) 
- 해당 방법을 이용하여 기존 리소스 또는 Docker Hub에 제공되지 않는 리소스(example: 사내 Agent)를 rootfs로 재가공하여 Concourse를 통해 사용 가능 할 것 같습니다.
- Opensource 방식의 Deploy에는 Custom Stack을 적용 가능하지만 PAS는 조금 힘들 것 같습니다. TAS를 변경하려면.. 내부 저장소에 Insert 해야 합니다.

## 1. rootfs 생성
- rootfs 생성에 대해 검색 해본 결과 scratch, binfmt-support 등 많은 Tool이 존재하지만 해당 문서는 Docker를 통해 rootfs를 생성 하였습니다.

### 1.1. Docker를 통한 rootfs 생성
- 사용 용도에 맞게 필요한 Package나 Binary를 사용 할 수 있는 Docker Image 생성 Dockerfile을 작성 합니다.
- 예시는 Java v1.8, Maven v3.6.1, CF7 Cli, Git를 기본적으로 사용 할 수 있게 구성 된 Dockerfile입니다.
```
# docker 이미지 생성
$ docker build --tag {IMAGE_NAME} .

# 생성한 Docker Image를 실행
$ docker run -i -t  {IMAGE_NAME} /bin/bash

# Image의 Container Id를 확인
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
8a9c250209f8        {IMAGE_NAME}      "/bin/bash"         33 seconds ago      Up 32 seconds                           awesome_jennings

# Container의 File System Export
docker export -o test.tgz {CONTAINER ID}

# 압축 파일을 해제하면 아래 Directory 구조가 완성 됩니다.
$ ls -al
total 84
drwxr-xr-x 21 root root 4096 Jun 22 18:13 .
drwxr-xr-x  3 root root 4096 Jun 22 19:03 ..
drwxr-xr-x  2 root root 4096 Jun 22 10:22 bin
drwxr-xr-x  2 root root 4096 Mar 28  2019 boot
drwxr-xr-x  4 root root 4096 Jun 22 14:16 dev
drwxr-xr-x 62 root root 4096 Jun 22 14:16 etc
drwxr-xr-x  2 root root 4096 Mar 28  2019 home
drwxr-xr-x  9 root root 4096 Jun 22 10:03 lib
drwxr-xr-x  2 root root 4096 Jun 10  2019 lib64
drwxr-xr-x  2 root root 4096 Jun 10  2019 media
drwxr-xr-x  2 root root 4096 Jun 10  2019 mnt
drwxr-xr-x  3 root root 4096 Jun 22 10:04 opt
drwxr-xr-x  2 root root 4096 Mar 28  2019 proc
drwx------  2 root root 4096 Jun 11  2019 root
drwxr-xr-x  4 root root 4096 Jun 22 10:04 run
drwxr-xr-x  2 root root 4096 Jun 22 10:03 sbin
drwxr-xr-x  2 root root 4096 Jun 10  2019 srv
drwxr-xr-x  2 root root 4096 Mar 28  2019 sys
drwxrwxrwt  3 root root 4096 Jun 22 10:22 tmp
drwxr-xr-x 10 root root 4096 Jun 10  2019 usr
drwxr-xr-x 11 root root 4096 Jun 10  2019 var

해당 폴더들을 rootfs가 최상위 디렉토리가 되도록 만들어줍니다.
$ mv -rf * ../rootfs/.

# metadata.json 편집 metadata는 Garden Runc에서 Container가 실행 될때 적용되는Conatiner Meta Data들입니다, 해당 예시에서는 java home과 binary 실행을 위해 PATH를 지정하였습니다, Disk의 할당량, 통계 등을 제공 할 수 있는 것 같습니다.
$ cat metadata.json
{"user":"root","env":["JAVA_HOME=/usr/local/openjdk-8","PATH=/usr/local/openjdk-8/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","HOME=/root"]}

# 최종 디렉토리 구조
$ ls -al
total 16
drwxr-xr-x  3 root   root   4096 Jun 22 19:03 .
drwxr-xr-x  6 ubuntu ubuntu 4096 Jun 22 19:05 ..
-rw-r--r--  1 root   root    164 Jun 22 18:36 metadata.json
drwxr-xr-x 21 root   root   4096 Jun 22 18:13 rootfs

# 최종 디렉토리를 통째로 압축 합니다.
$ tar cvzf test-0.0.1.tgz *
```

- 생성 된 test-0.0.1.tgz를 원격 저장 장소에 저장하고 Concourse Resource, Jobs, Task에서 사용 됩니다.
- fly hijack 명령어를 통해 test-0.0.1.tgz 가 사용 되고 있는 Container에 접속하면 Dockerfile에 명시한 java, maven, cf7 cli 등이 존재와 PATH 설정이 되어 있는 것을 확인 할 수 있습니다.
- rootfs 안에 파일을 생성하여 다시 압축 할 경우 생성한 파일을 Conatiner에서 확인 할 수 있습니다. 
