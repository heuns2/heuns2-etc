# Gemfire Partition Region Redundant 확인 방안
- Redundant 데이터는 Cache Server에 분산 된 Partition Region에 대해  한 Member가 상태 이상이 나타날 경우 Secondary Bucket에 존재하는 Data를 Primary Bucket으로 이동 시켜 데이터의 손실 없이 서비스가 가능 하도록 해줍니다.
- Redundant를 충족시킬 충분한 Copy 본이 없을 때마다 System은 다른 Member를 Secondary로 할당하고 여기에 Data를 복사하여 Redundant을 복구합니다.

## 1. 확인 방안

### 1.1. gfsh Metrics
- 정확하게 어떤 data가 어떤 cache server에 배치 되어 있는지 확인은 불가능 하지만 count를 통해서 예상 되는 Redundant의 Count 입니다.
- totalBucketSize(26) = primaryBucketCount(13) + secondryBucketCount(13)
- 아래 결과 중 numBucketsWithoutRedundancy가 1개 이상일 경우 reblance 작업이 필요 합니다.

```
$ show metrics --region=/leedh  
Cluster-wide Region Metrics  
          | primaryBucketCount           | 13  
          | numBucketsWithoutRedundancy  | 0  
          | totalBucketSize              | 26
```

### 1.2. gfsh function & command
- gfsh function과 command는 같은 동작을 key를 통해 어느 cache server에 해당 key가 Primary, Secondary에 배치되어 있는지에 대한 확인이 예상합니다.
- key 정보를 통하여 해당 key를 갖고 있는 entry 정보들의 위치를 확인 하는 방법
- Primary에서 *Primary PR* 값이 실제 data의 위치 *No* 영역이 Redundant의 위치
 
```
# example) Redundant 1개 일 경우
locate entry --key=testkey --region=/test
Result          : true
Key Class       : java.lang.String
Key             : testkeyleedh1
Locations Found : 2

                   MemberName                    |                                              MemberId                                               |   Primary    | BucketId
------------------------------------------------ | --------------------------------------------------------------------------------------------------- | ------------ | --------
cacheserver-f4f94bdf-xxxxx-xxxx-xxxx-xxxxx | f4f94bdf-xxxxx-xxxx-xxxx-xxxxx(cacheserver-f4f94bdf-xxxxx-xxxx-xxxx-xxxxx:8)<v65>:56152 | No           | 96
cacheserver-xxxx-8282-4e99-xxx-xxxx | xxxx-8282-4e99-xxx-xxxx(cacheserver-xxxx-8282-4e99-xxx-xxxx:8)<v69>:56152 | *Primary PR* | 96

# example) Redundant 2개 일 경우
locate entry --key=testkeyleedh1 --region=/test
Result          : true
Key Class       : java.lang.String
Key             : testkey1
Locations Found : 3

                   MemberName                    |                                               MemberId                                               |   Primary    | BucketId
------------------------------------------------ | ---------------------------------------------------------------------------------------------------- | ------------ | --------
cacheserver-f4f94bdf-xxxxx-xxxx-xxxx-xxxxx | f4f94bdf-xxxxx-xxxx-xxxx-xxxxx(cacheserver-f4f94bdf-xxxxx-xxxx-xxxx-xxxxx:8)<v65>:56152  | No           | 96
cacheserver-xxxx-8282-4e99-xxx-xxxx | xxxx-8282-4e99-xxx-xxxx(cacheserver-xxxx-8282-4e99-xxx-xxxx:8)<v69>:56152  | No           | 96
cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx | c410xxxx1bb1-xxx-4da3-8a03-xxxx(cacheserver-c410xxxx1bb1-xxx-4da3-8a03-xxxx:12)<v70>:56152 | *Primary PR* | 96
```
