# Gemfire VM Reblance
- Gfsh Reblance는 Server에 분산된 Region의 Data들을 재조정합니다.
- 구성된 파티션 영역 Redundancy가 충족되지 않는 경우 Rebalance은 Redundancy을 복구하기 위해 할 수있는 작업을 수행합니다. 
- Reblance는 Cluster 전체에서 데이터와 동작의 가장 공정한 균형을 설정하기 위해 필요에 따라 Host Member 간에 Partition Region 간의 Data Bucket를 이동합니다.

## 사용 방안
- Redundancy 설정을 하였지만 totalbucket count가 primary bucket count와 같은 경우
- Redundancy를 하였지만 gfsg - show metrics --region를 통한 numBucketsWithoutRedundancy의 결과가 0이 아닐 경우
- persistence disk 사용 시 cache-server 별 disk usage가 불균형일 경우 (특정 region의 data가 특정 server에만 쌓였는지 확인이 필요)

1) rebalance --simulate를 통해 reblance가 필요 한지 확인
2) rebalance --region을 통해 필요한 region 재조정  

## 특이 사항
- 실제 data가 저장되는 Bucket이 이동함에 따라 차단 될 확률이 있으며 Thread가 Disconnect 될 수 있습니다. 그로인해 부하가 낮거나 사용이 적은 시간에 Reblance를 실행 하여야 합니다.
- 주로 확인해야 할 metrics은 numBucketsWithoutRedundancy, Total primaries transferred during this rebalance, Total bytes in buckets moved during this rebalance입니다. 
