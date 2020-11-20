# chroot을 사용한 기본적인 CI/CD Script

- chroot는 Linux OS의 기본적인 Package이며 Namespace와 함께 Docker의 동작 원리입니다.

```
#/bin/bash
uuid=$(uuidgen)

echo $uuid

mkdir -p $uuid/lib/x86_64-linux-gnu/ $uuid/lib64 $uuid/bin $uuid/usr/local/bin/ $uuid/workspace/ $uuid/etc/ $uuid/tmp/

echo "password" | sudo -S cp /bin/bash $uuid/bin/

echo "password" | sudo -S cp manifest.yml $uuid/workspace/.
echo "password" | sudo -S cp spring-music.jar $uuid/workspace/.

echo "password" | sudo -S cp /usr/local/bin/cf $uuid/usr/local/bin/

echo "password" | sudo -S cp /etc/resolv.conf $uuid/etc/

echo "password" | sudo -S cp /lib/x86_64-linux-gnu/libtinfo.so.5 $uuid/lib/x86_64-linux-gnu/
echo "password" | sudo -S cp /lib/x86_64-linux-gnu/libdl.so.2 $uuid/lib/x86_64-linux-gnu/
echo "password" | sudo -S cp /lib/x86_64-linux-gnu/libc.so.6 $uuid/lib/x86_64-linux-gnu/
echo "password" | sudo -S cp /lib64/ld-linux-x86-64.so.2 $uuid/lib64

echo "password" | sudo -S chroot $uuid/ /bin/bash -c 'cf -v'

echo "password" | sudo -S chroot $uuid/ /bin/bash -c 'cf login -a https://api.sys.${DOMAIN} --skip-ssl-validation -u admin -p password -o system -s system'

echo "password" | sudo -S chroot $uuid/ /bin/bash -c 'cf push my-test-app -f workspace/manifest.yml -p workspace/spring-music.jar'


echo "password" | sudo -S  rm -rf $uuid

```
