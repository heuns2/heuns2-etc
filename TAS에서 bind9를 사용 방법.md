
# TAS에서 bind9를 사용 방법

## 1. PAS 배포 중 사용하는 PAS Domain의 종류는 크게 아래와 같다.
- api.{system_doamin}
- uaa.{system_doamin}
- ssh.{system_doamin}
- doppler.{system_doamin}
- apps.{system_doamin}
- login.{system_doamin}
- tcp.{system_doamin}

## 2. bind9 설치
```
$ sudo apt-get install bind9
```

### 3. bind9 설정 파일 위치
```
/etc/bind/*
```

### 4. bind9을 통하여 domain 연동
#### 4.1. bind9에 PAS에 사용할 Domain과 haproxy의 IP 또는 Router IP를 Mapping
```
$ cat /etc/bind/db.${DOMAIN}
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ${DOMAIN}. root.${DOMAIN}. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ${DOMAIN}.
@       IN      A       127.0.0.1
apps.sys            IN      A  ${SOURCE_IP}
uaa.sys              IN      A  ${SOURCE_IP}
logs.sys             IN      A  ${SOURCE_IP}
login.sys            IN      A  ${SOURCE_IP}
doppler.sys       IN      A  ${SOURCE_IP}
api.sys                IN      A  ${SOURCE_IP}
ssh.sys                IN      A  ${SOURCE_IP}
*.apps.sys          IN      A  ${SOURCE_IP}
*.apps                IN      A  ${SOURCE_IP}
*.sys                    IN      A  ${SOURCE_IP}
@                          IN      AAAA    ::1
```

#### 4.2. bind9 service가 start 될 때 db.${DOMAIN}이 적용 되도록 설정
```
cat /etc/bind/named.conf.local
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "${DOMAIN}" {
   type master;
   file "/etc/bind/db.${DOMAIN}";
};
```

#### 4.3. bind9 server restart
```
$ sudo service bind9 restart
$ sudo service bind9 status
● bind9.service - BIND Domain Name Server
   Loaded: loaded (/lib/systemd/system/bind9.service; enabled; vendor preset: enabled)
  Drop-In: /run/systemd/generator/bind9.service.d
           └─50-insserv.conf-$named.conf
   Active: active (running) since Tue 2019-07-23 07:57:44 UTC; 29min ago
     Docs: man:named(8)
  Process: 5279 ExecStop=/usr/sbin/rndc stop (code=exited, status=0/SUCCESS)
 Main PID: 5285 (named)
   CGroup: /system.slice/bind9.service
           └─5285 /usr/sbin/named -f -u bind

Jul 23 07:57:44 ubuntu named[5285]: configuring command channel from '/etc/bind/rndc.key'
Jul 23 07:57:44 ubuntu named[5285]: command channel listening on 127.0.0.1#953
Jul 23 07:57:44 ubuntu named[5285]: managed-keys-zone: loaded serial 2
Jul 23 07:57:44 ubuntu named[5285]: zone 0.in-addr.arpa/IN: loaded serial 1
Jul 23 07:57:44 ubuntu named[5285]: zone 127.in-addr.arpa/IN: loaded serial 1
Jul 23 07:57:44 ubuntu named[5285]: zone 255.in-addr.arpa/IN: loaded serial 1
Jul 23 07:57:44 ubuntu named[5285]: zone ${DOMAIN}/IN: loaded serial 2       ######### 적용 됨을 확인 #############
Jul 23 07:57:44 ubuntu named[5285]: zone localhost/IN: loaded serial 2
Jul 23 07:57:44 ubuntu named[5285]: all zones loaded
Jul 23 07:57:44 ubuntu named[5285]: running
```

#### 4.4. Domain이 bind9가 동작하고 있는 서버에 적용 되었는지 확인
```
$ dig @localhost apps.sys.${DOMAIN}

; <<>> DiG 9.10.3-P4-Ubuntu <<>> @localhost apps.sys.${DOMAIN}
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24328
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;apps.sys.${DOMAIN}.    IN      A

;; ANSWER SECTION:
apps.sys.${DOMAIN}. 604800 IN   A       ${SOURCE_IP} ###################### haprxoy 또는 Router IP를 확인

;; AUTHORITY SECTION:
${DOMAIN}.      604800  IN      NS      ${DOMAIN}.

;; ADDITIONAL SECTION:
${DOMAIN}.      604800  IN      A       127.0.0.1
${DOMAIN}.      604800  IN      AAAA    ::1

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Tue Jul 23 08:28:21 UTC 2019
;; MSG SIZE  rcvd: 129

```


### 5. Ops Manager Bosh Director Tile Config 수정
#### 5.1. Create Network Config에 DNS 설정 칸에 bind9이 실행 중인 IP 주소 입력

### 6. Apply Change
#### 6.1. 전체 Product Apply Change
