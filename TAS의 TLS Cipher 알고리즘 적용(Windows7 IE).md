#  TAS의 TLS Cipher 알고리즘 적용(Windows7 IE)
- 환경
	- Windows 7
	- IE 11.0..9

## 1. 문제점
- GO Router 구성 되어 있는 PCF 환경에서 Windows Internet Explore에서 Running 중인 App에 접속이 불가능 한 현상이 발생 함, 하지만 HA Proxy로 구성 되어 있는 PCF의 App은 접근이 가능
- Internet Explore 화면의 에러 메세지 (Go Router 구성)

```
이 페이지를 표시할 수 없습니다.

고급] 설정에서 TLS 1.0, TLS 1.1 및 TLS 1.2를 켠 다음 **https://{apps.{SYSTEM_DOMAIN}}** 에 다시 연결해 보세요. 이 오류가 계속되는 경우 이 사이트는 지원되지 않는 프로토콜 또는 안전하지 않은 RC4 [(세부 정보를 위한 링크)](http://go.microsoft.com/fwlink/?LinkId=735074)와 같은 암호 그룹을 사용할 수도 있습니다. 사이트 관리자에게 문의하세요.
```

## 2. 예상 원인
- Go Router 실행 Config 정보

```
---
status:
  port: 8080
  user: router_status
  pass: xyzADzrkGLYQ9oMkoVocClMMVn2JKb


nats:
  - host: ${IP}
    port: 4222
    user: nats
    pass: "aUcNTpHRjmAbZt4IW69sqjM8T3St0e"

  - host: ${IP}
    port: 4222
    user: nats
    pass: "aUcNTpHRjmAbZt4IW69sqjM8T3St0e"

  - host: ${IP}
    port: 4222
    user: nats
    pass: "aUcNTpHRjmAbZt4IW69sqjM8T3St0e"


logging:
  syslog: vcap.gorouter
  level: info
  loggregator_enabled: true
  metron_address: "localhost:3457"
  disable_log_forwarded_for: false
  disable_log_source_ip: false

tracing:
  enable_zipkin: true

port: 80
index: 0
go_max_procs: -1
trace_key: 22
debug_addr: 127.0.0.1:17002
secure_cookies: false

access_log:
  file: /var/vcap/sys/log/gorouter/access.log

publish_start_message_interval: 30s
prune_stale_droplets_interval: 30s
droplet_stale_threshold: 120s
publish_active_apps_interval: 0s # 0 means disabled
suspend_pruning_if_nats_unavailable: false
prune_stale_tls_routes: false
route_latency_metric_muzzle_duration: 30s


disable_keep_alives: false
max_idle_conns: 49000
max_idle_conns_per_host: 100
frontend_idle_timeout: 900s
backends:
  max_conns: 500
  cert_chain: |
    -----END CERTIFICATE-----
  private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    -----END RSA PRIVATE KEY-----
oauth:
  token_endpoint: uaa.service.cf.internal
  client_name: "gorouter"
  client_secret: ecMpVSYiQXOF0OHZ_iDkkcmZ_KJsp4LK
  port: 8443
  skip_ssl_validation: true
  ca_certs: "/var/vcap/jobs/gorouter/config/certs/uaa/ca.crt"

routing_api:
  uri: http://routing-api.service.cf.internal
  port: 3000
  auth_disabled: false


drain_wait: 20s
healthcheck_user_agent: HTTP-Monitor/1.1
endpoint_timeout: 900s

start_response_delay_interval: 20s
load_balancer_healthy_threshold: 20s

balancing_algorithm: round-robin

enable_ssl: true

tls_pem:
- cert_chain: |
    -----BEGIN CERTIFICATE-----
    -----END CERTIFICATE-----
  private_key: |
    -----BEGIN RSA PRIVATE KEY-----
    -----END RSA PRIVATE KEY-----

client_cert_validation: request
min_tls_version: TLSv1.2
ssl_port: 443

disable_http: false

ca_certs: |2+

  -----BEGIN CERTIFICATE-----
  -----END CERTIFICATE-----

  -----BEGIN CERTIFICATE-----
  -----END CERTIFICATE-----

  -----BEGIN CERTIFICATE-----
  -----END CERTIFICATE-----


skip_ssl_validation: true
cipher_suites: ECDHE-RSA-AES128-GCM-SHA256:TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
forwarded_client_cert: forward

isolation_segments: []
routing_table_sharding_mode: all

route_services_timeout: 60s
route_services_secret: YVHopeeqHM3L1iHTGLhR9kUf2wJuZ7
route_services_secret_decrypt_only:
route_services_recommend_https: true
route_services_hairpinning: true

extra_headers_to_log: []
token_fetcher_max_retries: 3
token_fetcher_retry_interval: 5s
token_fetcher_expiration_buffer_time: 30
enable_proxy: false
force_forwarded_proto_https: false
sanitize_forwarded_proto: false

http_rewrite: {"responses":{"add_headers_if_not_present":null,"remove_headers":[]}}

pid_file: /var/vcap/sys/run/gorouter/gorouter.pid

ip_local_port_range: 1024 65535
```

- HA Proxy Config 정보

```
global
    log /dev/log syslog info
    daemon
        user vcap
    group vcap
    maxconn 64000
    spread-checks 4
    tune.ssl.default-dh-param 2048
    tune.bufsize 16384
    stats socket /var/vcap/sys/run/haproxy/stats.sock mode 600 level admin
    stats timeout 2m
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    ssl-default-bind-ciphers DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    ssl-default-server-ciphers DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384

defaults
    log global
    option log-health-checks
    option log-separate-errors
    maxconn 64000
    option http-server-close
    option httplog
    option forwardfor
    option contstats
    timeout connect         5000ms
    timeout client          900000ms
    timeout server          900000ms
    timeout tunnel          3600000ms
    timeout http-keep-alive 900000ms
    timeout http-request    5000ms
    timeout queue           30000ms



listen health_check_http_url
    bind :8080
    mode http
    monitor-uri /health
    acl http-routers_down nbsrv(http-routers) eq 0
    monitor fail if http-routers_down


# HTTP Frontend {{{
frontend http-in
    mode http
      bind :80
    capture request header Host len 256
    default_backend http-routers
    acl xfp_exists hdr_cnt(X-Forwarded-Proto) gt 0
    reqadd X-Forwarded-Proto:\ http if ! xfp_exists
    acl ssl_redirect hdr(host),lower,map_end(/var/vcap/jobs/haproxy/config/ssl_redirect.map,false) -m str true
    redirect scheme https code 301 if ssl_redirect
# }}}

# HTTPS Frontend {{{
frontend https-in
    mode http
      bind :443  ssl crt /var/vcap/jobs/haproxy/config/ssl
    http-request del-header X-Forwarded-Client-Cert
    capture request header Host len 256
    default_backend http-routers
    acl xfp_exists hdr_cnt(X-Forwarded-Proto) gt 0
    reqadd X-Forwarded-Proto:\ https if ! xfp_exists
# }}}


# Default Backend {{{
backend http-routers
    mode http
    balance roundrobin
        errorfile 503 /var/vcap/jobs/haproxy/errorfiles/custom503.http
        server node0 172.18.81.25:80 crt /var/vcap/jobs/haproxy/config/backend-crt.pem check inter 1000
    server node1 172.18.81.26:80 crt /var/vcap/jobs/haproxy/config/backend-crt.pem check inter 1000
    server node2 172.18.81.27:80 crt /var/vcap/jobs/haproxy/config/backend-crt.pem check inter 1000
# }}}

# Routed Backends {{{
# }}}

# TCP Routing  {{{

frontend cf_tcp_routing
    mode tcp
    bind :1024-1123
    default_backend cf_tcp_routers

backend cf_tcp_routers
    mode tcp
    option httpchk GET /health

# TCP Backends
frontend tcp-frontend_diego_brain
    mode tcp
    bind :2222
    default_backend tcp-diego_brain

backend tcp-diego_brain
    mode tcp
    server node0 ${IP}:2222 check inter 1000
    server node1 ${IP}:2222 check inter 1000
    server node2 ${IP}:2222 check inter 1000


# }}}

- PCF의 Ha Proxy는 SSL 통신을 사용하여 IE에서 접근이 가능 추측 되지만 Go Router는 IE에서 TLS Version이 맞지 않으며 TLS의 Ciper 알고리즘이 틀려서 에러가 발생한 것으로 생각 된다. 
```
## 3. 에러 로그 및 현상 해결
- Go Router의 Error Log

```
2019/11/14 10:29:06 http: TLS handshake error from ${IP}:53639: tls: unsupported SSLv2 handshake received
2019/11/14 10:31:56 http: TLS handshake error from ${IP}:49598: tls: no cipher suite supported by both client and server
```
- PAS의 [Networking Config] -> [TLS Version]을 1.0으로 변경, [TLS cipher suites for Router]에 1.0 Version의 TSL Ciper 알고리즘 TLS_ECDHE_ECDSA_WITH_RC4_128_SHA:TLS_ECDHE_RSA_WITH_RC4_128_SHA:TLS_RSA_WITH_RC4_128_SHA추가

## 관련 링크
- https://docs.pivotal.io/platform/application-service/2-7/adminguide/securing-traffic.html#ciphers
- https://support.microsoft.com/ko-kr/help/3151631/rc4-cipher-is-no-longer-supported-in-internet-explorer-11-or-microsoft
- [https://www.securesign.kr/guides/kb/49](https://www.securesign.kr/guides/kb/49)
- [https://www.securesign.kr/guides/kb/16](https://www.securesign.kr/guides/kb/16)  <- 가장 많은 힌트.
