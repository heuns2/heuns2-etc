
# vSphere 물리 장비(Hardware)의 Spec 추출
- vSphere 상에 모든 Datastore의 Host Hardware 정보를 추출 한다.

## 1. 준비 사항
- vSphere govc cli
- vSphere 계정 정보
	- vSphere API endpoint(url)
	- vSphere username
	- vSphere password

## 2. Bash Shell
```
#!/bin/bash
export GOVC_URL={vShpere URL} # ex) 172.xxx.xx.x
export GOVC_USERNAME={vSphere Username} # ex) admin
export GOVC_PASSWORD={vSphere Password} # ex) 1q2w3e4r5t
export GOVC_INSECURE=true

datacenter_length=$(govc datacenter.info -json=true | jq '.Datacenters | length')

for (( j = 0 ; j < ${datacenter_length} ; j++))
do
        datastore_name=$(govc datacenter.info -json=true | jq '.Datacenters['$j'].Name' | tr -d '"')

        read -a cluster_array <<<$(govc ls ${datastore_name}/host)

        echo "===========================START================================="
        echo "****Datastore name = $datastore_name"

                for(( k = 0 ; k < ${#cluster_array[@]} ; k++ ))
                do
                        read -a host_array <<<$(govc ls ${cluster_array[$k]} | grep -v "Resource")
                        for (( i = 0 ; i < ${#host_array[@]} ; i++))
                        do
                          govc host.info -dc=${datastore_name} -host=${host_array[$i]} -json | \
                          jq '.HostSystems[0] | {
                          HostName: .Name,
                          CpuCores: .Summary.Hardware.NumCpuCores,
                          CpuGhz: .Summary.Hardware.CpuMhz,
                          Socket: .Summary.Hardware.NumCpuPkgs,
                          CpuModel: .Summary.Hardware.CpuModel,
                          LogicalProcess: .Summary.Hardware.NumCpuThreads,
                          MemorySize: .Summary.Hardware.MemorySize
                          }'
                        done
                done

        echo "===========================STOP================================="
done
```
## 3. Bash Shell 결과 값 일부 추출
- Cpu GHz = 12 x 3.392
- Memory = 274594.250752 MB
- Core/Socket = 12/2 result 6
```
===========================START=================================
****Datastore name = {datastore_name}
{
  "HostName": "{hostName}",
  "CpuCores": 12,
  "CpuGhz": 3392,
  "Socket": 2,
  "CpuModel": "Intel(R) xxxx CPU xxxxx v3 @ xxxxxx",
  "LogicalProcess": 24,
  "MemorySize": 274594250752
}
===========================END=================================
```

- 추가적으로 추출할 데이터가 있을 경우 govc host.info -dc={DATACENTER_NAME} -host={HOST_NAME}를 이용하여 결과 값을 출력하고 Json Key 값을 MemorySize: .Summary.Hardware.MemorySize 아래에 추가하여 Script를 실행하여 Hardware Spec을 확인한다.
