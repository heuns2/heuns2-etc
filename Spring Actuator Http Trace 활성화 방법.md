
# Spring Actuator Http Trace 활성화 방법

- Spring Boot v2.2.1 이후 부터는 과도한 inmemory 사용으로 인하여 httptrace를 default로 disable 시켰습니다.
https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.2-Release-Notes#actuator-http-trace-and-auditing-are-disabled-by-default

 httptrace를 다시 활성화 시키려면 약간의  소스 코드 개발이 필요 합니다.

- application.yml

```
management:
   security:
     enabled: false
   trace:
     http:
       enabled: true
       include: remote_address, request_headers, response_headers
   auditevents:
     enabled: true
   cloudfoundry:
     skip-ssl-validation: true
   endpoints:
     web:
       exposure:
         include: '*'
```

- HttptraceConfig.java

```
package com.boot.test;

import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedList;
import java.util.List;

import org.springframework.boot.actuate.trace.http.HttpTrace;
import org.springframework.boot.actuate.trace.http.HttpTraceRepository;
import org.springframework.context.annotation.Configuration;

@Configuration
public class HttpTraceConfig implements HttpTraceRepository {
	


    private int capacity = 100; // 보여질 httptrace의 총 수
    private boolean reverse = true;
    private final List<HttpTrace> traces = new LinkedList<>();

    public void setReverse(boolean reverse) {
        synchronized (this.traces) {
            this.reverse = reverse;
        }
    }

    public void setCapacity(int capacity) {
        synchronized (this.traces) {
            this.capacity = capacity;
        }
    }

    @Override
    public List<HttpTrace> findAll() { 
        synchronized (this.traces) {
            return Collections.unmodifiableList(new ArrayList<>(this.traces)); // 전체 http trace를 배열에 담아서 리턴
         }
    }

    @Override
    public void add(HttpTrace trace) {  // http 요청을 추가
        synchronized (this.traces) {
            while (this.traces.size() >= this.capacity) {
                this.traces.remove(this.reverse ? this.capacity - 1 : 0);
            }
            if (this.reverse) {
                this.traces.add(0, trace);
            }
            else {
                this.traces.add(trace);
            }
        }
    }
}

```
