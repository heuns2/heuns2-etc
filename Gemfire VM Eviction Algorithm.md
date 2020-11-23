# Gemfire VM Eviction Algorithm
- Gemfire Eviction은 Heap Memory 사용률이 특정 임계치(75%)를 초과하면 Algorithm 유형에 따라 Region Data를 퇴거 시키는 속성입니다.
- Gemfire Eviction의 종류로는 세가지가 존재하며 현재 Eviction을 Custom하게 사용 할 수 없습니다.

## 1. Eviction 알고리즘 종류
- Eviction의 알고리즘 종류로는 3가지가 존재하고 있습니다. 유형 별로 Cache-Server에 종속, Region 별 종속으로 나누어집니다.

1) lru-heap-percentage: Server에 종속되며 Server Heap Memory 75% 기준으로 Eviction이 발생합니다. 이때 기준은 Round Robin 방식으로 Region의 가장 오래 된 항목(LRU)을 삭제 합니다. 
2) absolute memory usage: Region에 종속되며 Size를 비교하여 비슷한 Size를 Eviction 합니다.
3) entry count: Region에 종속되며 이전 Entry를 1:1 형태로 Eviction 합니다.

## 2. Eviction Type
- Eviction이 발생하게 되면  2가지 유형으로 Data가 관리 됩니다.

1) Local-Destory: Algorithm 유형에 맞는 Entry를 바로 삭제
2) Persistence Disk Overflow:  Algorithm 유형에 적합한 Entry를 Disk에 Entry Value가 Persistence Disk 영역에 Overflow 됩니다. 

## 3. Eviction에 대한 특이 사항
- lru-heap-percentage는 Disk Overflow로 Region을 생성 할 경우 Default로 사용되는 전역 변수입니다. 만약 해당 속성을 사용하지 않고 다른 알고리즘을 사용 할 경우 Create Region에 대한 속성을 재 정의 해야 합니다.
- absolute memory usage 알고리즘을 사용 할 경우 Partition, Replica 속성에 따라 create region시 다른 option이 사용 되어야 합니다.

```
Partition Type일 경우: --local-max-memory를 사용
Replica Type일 경우: --eviction-max-memory를 사용
```

