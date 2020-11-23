# Siege Tool 사용 (Http Load Test Tool)

- Siege는 HTTP로드 테스트 및 벤치마킹 유틸리티로, 협박시 웹 서버의 성능을 측정하는 데 사용할 수 있습니다. 전송 된 데이터의 양, 서버의 응답 시간, 트랜잭션 속도, 처리량, 동시성 및 프로그램이 올바로 리턴 한 시간을 평가합니다.

## Siege 설치 방법 & 설정

### 1. Siege 설치 
```
# siege 다운로드
$ wget http://download.joedog.org/siege/siege-latest.tar.gz
$ tar -zxvf siege-latest.tar.gz
$ cd siege-*/
$ ./configure
$ make
$ sudo make install
```

### 2. Siege  설정
- Seige가 설치 완료 되면 config 파일은 home 디렉토리의 .siege/siege.conf 파일에 위치합니다.
- 사용자 설정 Siege 측정에 사용 될 사용자를 명시하는 곳은 config 파일의 아래 속성입니다.
```
concurrent = 25 # 25명
```

### 3. Siege 실행 방법 및 결과 정리
#### 3.1. Siege 실행

```
$ siege {APP_DOMAIN}/simulate/delay/1

# 결과 값
{       "transactions":                         1995, # Server에 대한 transaction 수
        "availability":                       100.00, # 서버에서 처리한 소켓 연결의 백분율
        "elapsed_time":                         1.11, # 전체 Siege 시간
        "data_transferred":                     0.03, # 전송 된 데이터의 합계
        "response_time":                        0.01, # 응답 시간
        "transaction_rate":                  1797.30, # transaction 비율
        "throughput":                           0.03, # 처리량
        "concurrency":                         24.75, # 서버에 동시 연결 수
        "successful_transactions":              1995, # 서버가 return code 400 미만으로 응답한 수
        "failed_transactions":                     0, # 서버가 return code 400 이상으로 응답한 수
        "longest_transaction":                  0.04, # 단일 transaction에서의 소요 최대 시간
        "shortest_transaction":                 0.01 # 단일 transaction에서의 소요 최소 시간
}
```
