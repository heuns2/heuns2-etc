
# Bosh Deploy 중 VM의 AZ 배치 분배 규칙
- PCF 가용존을 Cloud Config를 통해 az1, az2를 사용하게 하였을 경우 azs 1번째(az1) Cluster의 사용량이 100%일 경우 azs 2번째(az2)로 설치가 진행이 되는지  (실제 물리 장비가 있는 Private IaaS가 적합)


## 1. Bosh의 Deployment Azs 할당 규칙
- 새로운 Instance는 지정된 AZ에 최대한 고르게 분산 된다.
- 기존 Instance는 가능한 경우 유지되지만 배포를 균일하게하기 위해 필요한 경우 재조정된다.
- Persistence Disk가있는 기존 Persistence Disk 데이터 손실을 피하기 위해 재조정되지 않는다.
- 제거 된 AZ의 기존 Instance가 제거되고 Persistence Disk가 분리된다.
- Static IP가 하나 이상의 Network에 지정된 경우 AZ 선택은 IP의 AZ 할당을 충족시키기 위해 집중 된다.

## 2. PAS의 HA
- 만약 AZ가 3개로 구성 되어 있을 경우 1개의 AZ가 내려갔을 경우 정상 작동.
- 만약 AZ가 5개로 구성 되어 있을 경우 2개의 AZ가 내려갔을 경우 정상 작동.
- 약 60~70%의 AZ가 살아 있으면 PAS System은 정상동작 하는 것 같다.

## 3. Bosh Deploy 중 VM의 AZ 배치 분배 규칙 결과

### 3.1. 신규 Instance 배치 예시
- 예시) Deployment 구성도
	`az1: 2 Instance 배치`
	`az2: 2 Instance 배치`
- 위 상황에서 추가적으로 2개의 Instance을 배포 시 Bosh는 az1에 Instance 1개를, az2에 Instance 1개를 균일하게 배포 한다.
- 만약 이상황에서 az1의 Cluster가 Full일 경우에 새로운 Instance 추가에 대한 배포가 실패한다.
- 사전에 미리 vCluster를 모니터링해야 하며 용량이 부족하다면 사전에 vCluster, vHost를 추가 구성 해야 한다
- 추가적으로 Instance의 재조정 문제는 추 후의 계획에 Update 되는 bosh cli의 rebalance 옵션을 통해 가능할 것으로 생각 된다.
