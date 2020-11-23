
# Tanzu Application Service Java App 한글 깨짐 Issue

## 1. 현상
- Local Windows 환경에서는 한글에 대한 문제가 존재하지 않지만 cflinuxfs (Ubuntu Linux)에서는 한글 글꼴이 없어 Java에서 File 관리 시 한글이 깨져서 나오는(네모네모네모) 현상이 발견 되었습니다.
- 조금 더 명확하게는 StringReader와 BufferedImage, InputStream에 대해 File Format -> String -> Image -> Stream 변환 간 파일 한글 깨짐 현상이 발견 되었습니다.


## 2. 해결 방법
- 해결 방법은 2가지에 대하여 2가지 방법이 있다고 생각 됩니다.

### 2.1. Cflinuxfs Rootfs 추가
- Cflinuxfs는 Stemcell OS 기반으로 돌아가게 되는 Filesystem이며 grootfs를 통하여 Container화 관리 됩니다.
- 하지만 Tanzu Platform에서 Custom Rootfs 사용은 거의 불가능하며 runtime config를 통하여 diego_cell의 특정 rootfs 디렉토리(diego cell VM)에 항상 배치 및 인증서 배치 등 작업이 필요 합니다.

### 2.2. Java Buildpack 변경
- 해당 방법으로 이슈를 해결 하긴 했지만 JRE 버전과 Java 버전으로 인하여 해결에 대한 약간 상이한 부분이 있을 수 있습니다.
- Java JRE의 수정 사항

```
# jre directory 구조
$ ls -al
drwxr-xr-x  2 uucp  143   4096 Dec 16  2018 bin
-r--r--r--  1 uucp  143   3244 Dec 16  2018 COPYRIGHT
drwxr-xr-x 15 uucp  143   4096 Dec 16  2018 lib
-r--r--r--  1 uucp  143     40 Dec 16  2018 LICENSE
drwxr-xr-x  4 uucp  143   4096 Dec 16  2018 man
drwxr-xr-x  3 uucp  143   4096 Dec 16  2018 plugin
-r--r--r--  1 uucp  143     46 Dec 16  2018 README
-rw-r--r--  1 uucp  143    424 Dec 16  2018 release
-rw-r--r--  1 uucp  143 112724 Dec 12  2018 THIRDPARTYLICENSEREADME-JAVAFX.txt
-r--r--r--  1 uucp  143 153824 Dec 16  2018 THIRDPARTYLICENSEREADME.txt
-r--r--r--  1 uucp  143    955 Dec 16  2018 Welcome.html

# /lib/fonts로 이동 & fonts 디렉토리 구조 확인
$ ls -al
-rw-r--r--  1 uucp 143   4122 Jul 13 08:38 fonts.dir
-rw-r--r--  1 uucp 143  75144 Dec 16  2018 LucidaBrightDemiBold.ttf
-rw-r--r--  1 uucp 143  75124 Dec 16  2018 LucidaBrightDemiItalic.ttf
-rw-r--r--  1 uucp 143  80856 Dec 16  2018 LucidaBrightItalic.ttf
-rw-r--r--  1 uucp 143 344908 Dec 16  2018 LucidaBrightRegular.ttf
-rw-r--r--  1 uucp 143 317896 Dec 16  2018 LucidaSansDemiBold.ttf
-rw-r--r--  1 uucp 143 698236 Dec 16  2018 LucidaSansRegular.ttf
-rw-r--r--  1 uucp 143 234068 Dec 16  2018 LucidaTypewriterBold.ttf
-rw-r--r--  1 uucp 143 242700 Dec 16  2018 LucidaTypewriterRegular.ttf

# fonts 디렉토리에서 fallback 디렉토리를 생성
$ mkdir fallback

# fallback 디렉토리에 적용 할 한글 폰트를 이동
$ ls -al
-rw-r--r-- 1 uucp 143 4186060 Jul 13 09:12 NanumBarunGothic.ttf

$ fonts 디렉토리의 fonts.dir 맨 아래에 Line 추가
NanumBarunGothic.ttf -ms-nanumgothicgothic-medium-r-normal--0-0-0-0-c-0-ksc5601.2017-0

# JRE 폴더를 전체 tar.gz로 압축
$ tar -zcvf jre-8u202-linux-x64.tar.gz jre1.8.0_202/

# 해당 JRE를 기준으로 신규 Java Buildpack 생성
```

- 위 수정을 진행하고 buildpack을 다시 Packaging 하게 되면 한글이 올바르게 출력 되는 것을 확인 할 수 있습니다.
