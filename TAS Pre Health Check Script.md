```
#!/bin/bash
om --username='admin' --password='admin' --target='https://opsmanager.{DOMAIN}' -k bosh-env > bosh-target.sh
om --username='admin' --password='admin' --target='https://opsmanager.{DOMAIN}' -k curl -p /api/v0/deployed/products > deployed_products.json

source ./bosh-target.sh
bosh_target_info=$(bosh env)

##################################
#1. bosh target 정보 검사
##################################
if [[ $bosh_target_info = "" ]]; then
    echo "##################################wrong bosh director info ##################################"
    exit 1;
else
    echo "bosh target info:" $bosh_target_info
fi

##################################
#2. platform 만료 인증서 검사 (6개월)
##################################

echo "##################################certificate expiration check start##################################"
om --username='admin' --password='admin' --target='https://opsmanager.{DOMAIN}' -k \
curl -p /api/v0/deployed/certificates?expires_within=6m \
| jq .certificates[] > expired_cert_check.json

expired_cert_check=$(cat ./expired_cert_check.json)

if [[ $expired_cert_check != "" ]]; then
    echo $expired_cert_check
else
    echo "There are no certificates that will expire in the next 6 months"
fi

echo "##################################certificate expiration check start##################################"


##################################
#3. platform instance 상태 검사
##################################
echo "##################################platform instance status check start##################################"

bosh is --ps > bosh-instance

vm_status=$(cat ./bosh-instance  | awk '{print $1,$2,$3}' | grep -v bosh-health-check |grep -e failing -e unresponsive)
if [[ $vm_status != "" ]]; then
    echo "require vms status check "
    echo $vm_status
    exit 1;
fi

# VM은 생성 되었지만 Job이 안뜬 상태로 남아 있는 VM 검출, ondemand VM이 존재 할 수 있음으로 눈으로 확인 필요
cat ./bosh-instance | awk '{print $1}' > monit_status
vm_monit_uniq_vms=$(awk '{!vms[$0]++};END{for(i in vms) if(vms[i]==1)print i}' monit_status  | grep -v 'deploy-all\|delete-all\|deploy-service-broker\|destroy-broker')

if [[ $vm_monit_uniq_vms != "" ]]; then
    echo $vm_monit_uniq_vms
else
    echo "vm jobs monit check success"
fi


echo "##################################platform instance status check end##################################"

########################
#4. BOSH TASK & 설정 검사
########################
echo "##################################bosh task check start##################################"

# 최근 100개의 task 중 Error가 존재 하는지 확인
om --username='admin' --password='admin' --target='https://opsmanager.{DOMAIN}' -k \
curl -p /api/v0/diagnostic_report | jq .director_configuration.resurrector_enabled > resurrector_config_check.json

resurrector_config_check=$(cat resurrector_config_check.json)

if [[ $resurrector_config_check != true ]]; then
    echo "bosh resurrector: " $resurrector_config_check
else
    echo "bosh resurrector: " $resurrector_config_check "If set to false, the VM does not automatically survive."
fi


current_bosh_task_info=$(bosh tasks | awk '{print $1 ,$2, $10, $11}')
# bosh가 진행하고 있는 task process가 존재하는지, queue 상태가 있는지 확인
if [[ $current_bosh_task_info != "" ]]; then
    echo $current_bosh_task_info
else
    echo "no processing bosh task"
fi

# 최근 100개의 task 중 Error가 존재 하는지 확인
lately100_bosh_task_info=$(bosh tasks -a -r=100 | awk '{print $1 ,$2, $15, $16, $17, $18}' | grep -e error -e fail)
if [[ $lately100_bosh_task_info != "" ]]; then
    echo $lately100_bosh_task_info
else
    echo "no error lately 100 bosh task"
fi

echo "##################################bosh task check end##################################"

########################
#5. 전체 Platform VM CPU/Disk/Memory 용량 확인 Ops Manager 제외
########################
echo "########################start platform VM usage check########################"

# PAS 정보 설정
cf_deployment_name=$(jq -r '.[] | select(.type == "cf") | .guid' "deployed_products.json")
bosh -d $cf_deployment_name manifest > cf_manifest.yml
cf_api_endpoint=$(cat cf_manifest.yml  | grep api_url | sed -n '1,1p' | awk '{print $2}')
system_domain=$(cat cf_manifest.yml  | grep system_domain | sed -n '1,1p' | awk '{print $2}')
cf_logcache_endpoint=$(curl $cf_api_endpoint -k  | jq .links.log_cache.href | tr -d '"')

# Credhub 정보 설정
credhub_api=$(echo $CREDHUB_SERVER)
credhub_client=$(echo $CREDHUB_CLIENT)
credhub_secret=$(echo $CREDHUB_SECRET)
bosh_ip=$(echo $BOSH_ENVIRONMENT)


echo "##################################bosh persistence disk usage check start##################################"
# Bosh persistence disk 사용량 검사
om --env ./env/$ENV_FILE \
credentials --product-name=p-bosh --credential-reference=.director.bbr_ssh_credentials --format=json \
| jq .private_key_pem | tr -d '"' > bbr_oneline_key.pem

sed -e "s/-----BEGIN RSA PRIVATE KEY-----/&\n/" \
    -e "s/-----END RSA PRIVATE KEY-----/\n&/" \
    -e "s/\S\{64\}/&\n/g" \
    ./bbr_oneline_key.pem | sed -e 's/\\n//g' > bbr_key.pem

chmod 400 ./bbr_key.pem
ssh -o StrictHostKeyChecking=no -i ./bbr_key.pem bbr@$bosh_ip "df -h" | grep "var/vcap/store" > bosh_persistence_disk_size.json
echo "bosh persistence disk usage: "$(cat ./bosh_persistence_disk_size.json)
disk_usage=$(cat ./bosh_persistence_disk_size.json | awk '{print $5}' | sed -e 's/%//g')

if [ $disk_usage -gt 80 ]; then
    echo "It is necessary to increase the bosh persistence disk capacity."
    exit 1;
fi

echo "##################################bosh persistence disk usage check end##################################"


credhub_access_token=$(curl -k "https://$bosh_ip:8443/oauth/token" \
-X POST -H 'Content-Type: application/x-www-form-urlencoded' \
-H 'Accept: application/json' \
-d 'client_id='$credhub_client'&client_secret='$credhub_secret'&grant_type=client_credentials'| jq .access_token | tr -d '"')

export TOKEN="$credhub_access_token"

cf_user_name=$(curl -k "$credhub_api/api/v1/data?name=/opsmgr/$cf_deployment_name/uaa/admin_credentials&current=true" \
-X GET -H 'Content-Type: application/json' \
-H "Authorization: Bearer $TOKEN" \
| jq .data[].value.identity | tr -d '"')


cf_user_password=$(curl -k "$credhub_api/api/v1/data?name=/opsmgr/$cf_deployment_name/uaa/admin_credentials&current=true" \
-X GET -H 'Content-Type: application/json' \
-H "Authorization: Bearer $TOKEN"  \
| jq .data[].value.password | tr -d '"')


cf login -a $cf_api_endpoint  -u $cf_user_name -p $cf_user_password -o system -s system --skip-ssl-validation


echo "##################################platform disk usage check start##################################"

# 전체 platform VM persistence disk 용량 검사 (80% 이상)
curl -G "$cf_logcache_endpoint/api/v1/query" \
--data-urlencode 'query=system_disk_persistent_percent{source_id="bosh-system-metrics-forwarder"} > 80' \
-k -H "Authorization: $(cf oauth-token)" | \
jq '[ .data.result[] | {
deployment_name:.metric.deployment,
job_name:.metric.job,
index:.metric.index,
usage: .value[1],
date: .value[0]
}]' > vm_persistence_check.json

# 전체 platform VM ephemeral disk 용량 검사 (80% 이상)
curl -G "$cf_logcache_endpoint/api/v1/query" \
--data-urlencode 'query=system_disk_ephemeral_percent{source_id="bosh-system-metrics-forwarder"} > 80' \
-k -H "Authorization: $(cf oauth-token)" | \
jq '[ .data.result[] | {
deployment_name:.metric.deployment,
job_name:.metric.job,
index:.metric.index,
usage: .value[1],
date: .value[0]
}]' > vm_ephemeral_check.json

vm_persistent_index=$(cat ./vm_persistence_check.json | jq length | tr -d '"')
vm_ephemeral_index=$(cat ./vm_ephemeral_check.json | jq length | tr -d '"')

if [ $vm_persistent_index -eq 0 ]; then
    echo "all VMs persistence disk usage 80% under"
else 
    for ((i=0;i<$vm_persistent_index;i++))
    do
      deployment_name=$(cat vm_persistence_check.json | jq .[$i].deployment_name | tr -d '"')
      job_name=$(cat vm_persistence_check.json | jq .[$i].job_name | tr -d '"')
      index_name=$(cat vm_persistence_check.json | jq .[$i].index | tr -d '"')
      usage=$(cat vm_persistence_check.json | jq .[$i].usage | tr -d '"')
      date=$(cat vm_persistence_check.json | jq .[$i].date | tr -d '"')
      if [[ $deployment_name != "bosh-health-check" ]]; then
        echo $date $job_name/$index_name "Persistent Disk Usage $usage%"
      fi
    done
fi

if [ $vm_ephemeral_index -eq 0 ]; then
    echo "all VMs ephemeral disk usage 80% under"
else 
    for ((i=0;i<$vm_ephemeral_index;i++))
    do
      deployment_name=$(cat vm_ephemeral_check.json | jq .[$i].deployment_name | tr -d '"')
      job_name=$(cat vm_ephemeral_check.json | jq .[$i].job_name | tr -d '"')
      index_name=$(cat vm_ephemeral_check.json | jq .[$i].index | tr -d '"')
      usage=$(cat vm_ephemeral_check.json | jq .[$i].usage | tr -d '"')
      date=$(cat vm_ephemeral_check.json | jq .[$i].date | tr -d '"')
      if [[ $deployment_name != "bosh-health-check" ]]; then
        echo $date $job_name/$index_name "Ephemeral Disk Usage $usage%"
      fi
    done
fi
echo "##################################platform disk usage check end##################################"


echo "##################################platform cpu usage check start##################################"
# 전체 platform VM CPU 사용률 검사 (80% 이상)
curl -G "$cf_logcache_endpoint/api/v1/query" \
--data-urlencode 'query=avg_over_time(system_cpu_user{source_id="bosh-system-metrics-forwarder"}[30s])+avg_over_time(system_cpu_sys{source_id="bosh-system-metrics-forwarder"}[30s])>80' \
-k -H "Authorization: $(cf oauth-token)" | \
jq '[ .data.result[] | {
deployment_name:.metric.deployment,
job_name:.metric.job,
index:.metric.index,
usage: .value[1],
date: .value[0]
}]' > vm_cpu_check.json

vm_cpu_usage_avg_indx=$(cat ./vm_cpu_check.json | jq length | tr -d '"')

if [ $vm_cpu_usage_avg_indx -eq 0 ]; then
    echo "all VMs CPU usage 80% under"
else
    for ((i=0;i<$vm_cpu_usage_avg_indx;i++))
    do
      deployment_name=$(cat vm_cpu_check.json | jq .[$i].deployment_name | tr -d '"')
      job_name=$(cat vm_cpu_check.json | jq .[$i].job_name | tr -d '"')
      index_name=$(cat vm_cpu_check.json | jq .[$i].index | tr -d '"')
      usage=$(cat vm_cpu_check.json | jq .[$i].usage | tr -d '"')
      date=$(cat vm_cpu_check.json | jq .[$i].date | tr -d '"')
      if [[ $deployment_name != "bosh-health-check" ]]; then
        echo "$date $job_name/$index_name CPU Usage $usage%"
      fi
    done
fi

echo "##################################platform cpu usage check end##################################"

echo "##################################platform memory usage check start##################################"
# 전체 platform VM Memory 사용률 검사 (80% 이상)
curl -G "$cf_logcache_endpoint/api/v1/query" \
--data-urlencode 'query=system_mem_percent{source_id="bosh-system-metrics-forwarder"} > 80' \
-k -H "Authorization: $(cf oauth-token)" | \
jq '[ .data.result[] | {
deployment_name:.metric.deployment,
job_name:.metric.job,
index:.metric.index,
usage: .value[1],
date: .value[0]
}]' > vm_memory_check.json

vm_memory_usage_avg_indx=$(cat ./vm_memory_check.json | jq length | tr -d '"')

if [ $vm_memory_usage_avg_indx -eq 0 ]; then
    echo "all VMs memory usage 80% under"
else 
    for ((i=0;i<$vm_memory_usage_avg_indx;i++))
    do
      deployment_name=$(cat vm_memory_check.json | jq .[$i].deployment_name | tr -d '"')
      job_name=$(cat vm_memory_check.json | jq .[$i].job_name | tr -d '"')
      index_name=$(cat vm_memory_check.json | jq .[$i].index | tr -d '"')
      usage=$(cat vm_memory_check.json | jq .[$i].usage | tr -d '"')
      date=$(cat vm_memory_check.json | jq .[$i].date | tr -d '"')
      if [[ $deployment_name != "bosh-health-check" ]]; then
        echo "$date $job_name/$index_name CPU Usage $usage%"
      fi
    done
fi
echo "##################################platform memory usage check end##################################"

echo "########################platform VM usage check end########################"

########################
#6. PAS Internal Mysql 상태 확인
########################
echo "########################PAS intern mysql health check start########################"

mysql_user_name=$(curl -k "$credhub_api/api/v1/data?name=/p-bosh/$cf_deployment_name/mysql-proxy-dashboard-credentials&current=true" \
-X GET -H 'Content-Type: application/json' \
-H "Authorization: Bearer $TOKEN" \
| jq .data[].value.username | tr -d '"')

mysql_user_password=$(curl -k "$credhub_api/api/v1/data?name=/p-bosh/$cf_deployment_name/mysql-proxy-dashboard-credentials&current=true" \
-X GET -H 'Content-Type: application/json' \
-H "Authorization: Bearer $TOKEN" \
| jq .data[].value.password | tr -d '"')


curl "https://0-proxy-p-mysql-ert.$system_domain/v0/backends" -u $mysql_user_name:$mysql_user_password -k > mysql_health_check.json
mysql_unhealthy=$(cat ./mysql_health_check.json | grep '"healthy":false')
if [[ $mysql_unhealthy != "" ]]; then
    echo $mysql_unhealthy
    echo "PAS Mysql Unhealthy"
    exit 1
else
    echo $(cat ./mysql_health_check.json)
fi
echo "########################PAS intern mysql health check end########################"

########################
#7. PAS Diego Cell 배포 공간 확인
# diego cell의 max_in_flight, memory, disk 사이즈 계산
# example) diego_cell Mem: 32GB, Disk 64GB ,max_in_flight가 5개일 경우 32GB*5 Memory, 64GB*5의 Disk의 공간이 있어야 무리 없이 upgrade가 가능 
########################

echo "########################diego capacity check start########################"

om --username='admin' --password='admin' --target='https://opsmanager.{DOMAIN}' \
-k curl -p /api/v0/staged/products/$cf_deployment_name/max_in_flight \
| jq . | grep diego_cell > diego_cell_max_in_flight.json

diego_job_name=$(cat ./diego_cell_max_in_flight.json | awk '{print $1}' | tr -d '"' | tr -d ":")
max_in_flight=$(cat ./diego_cell_max_in_flight.json | awk '{print $2}' | tr -d '"' | tr -d ",")

om --username='admin' --password='admin' --target='https://opsmanager.{DOMAIN}' -k \
curl -p /api/v0/staged/products/$cf_deployment_name/jobs/$diego_job_name/resource_config \
| jq . > diego_job_info.json


max_in_flight_check=$(echo $max_in_flight | grep %)
diego_cell_vm_type=$(cat diego_job_info.json | jq .instance_type.id)
diego_cell_instance_count=$(cat diego_job_info.json | jq .instances | tr -d '"')


om --username='admin' --password='admin' --target='https://opsmanager.{DOMAIN}' -k \
curl -p "/api/v0/staged/cloud_config" \
| jq '.cloud_config.vm_types[] | select(.name == '$diego_cell_vm_type')' > diego_cell_vm_size.json



current_diego_cell_disk_size=$(cat ./diego_cell_vm_size.json | jq .cloud_properties.disk  ) 
current_diego_cell_memory_size=$(cat ./diego_cell_vm_size.json | jq .cloud_properties.ram )

if [[ $max_in_flight_check != "" ]]; then
    max_in_flight_substring=${max_in_flight_check:0:1}
    percentage=$(printf '%.2f\n' "$(echo "scale=2;$max_in_flight_substring/100" | bc)")
    calculator=$(printf '%.2f\n' "$(echo "scale=2;$diego_cell_instance_count*$percentage" | bc)")
    j=0
    for i in $(echo $calculator | tr "." "\n")
    do
      if [ $j -eq 0 ]; then
        max_in_flight_Integer=$i
        j=$(($j+1))
        max_in_flight_result=$max_in_flight_Integer
      else 
        max_in_flight_decimal=$i
        if [ $max_in_flight_decimal -ne 0 ]; then
          max_in_flight_result=$(echo $max_in_flight_Integer+1 | bc -l)
        fi
      fi
    done
else
  max_in_flight_result=$max_in_flight
fi



least_diego_cell_disk_size=$(echo "$max_in_flight_result*$current_diego_cell_disk_size" | bc)
least_diego_cell_memory_size=$(echo "$max_in_flight_result*$current_diego_cell_memory_size" | bc)

# health watch를 사용하지 않을 경우 직접 계산 해야 함.
available_diego_cell_disk_size=$(echo "scale=2;$(curl -k -G "$cf_logcache_endpoint/api/v1/query" --data-urlencode 'query=Diego_AvailableFreeChunksDisk {source_id="healthwatch-forwarder",deployment="'$cf_deployment_name'"}[5m]' -H "Authorization: $(cf oauth-token)" | jq .data.result[].'values[]' |  jq .[1] | tr -d '"')*4096" | bc)
available_diego_cell_memory_size=$(echo "scale=2;$(curl -k -G "$cf_logcache_endpoint/api/v1/query" --data-urlencode 'query=Diego_AvailableFreeChunks {source_id="healthwatch-forwarder",deployment="'$cf_deployment_name'"}[5m]' -H "Authorization: $(cf oauth-token)" | jq .data.result[].'values[]' | jq .[1] | tr -d '"')*4096" | bc)


echo (1+1 | ./bc)

if [ $least_diego_cell_disk_size -gt $available_diego_cell_disk_size ]; then
    echo "max_in_flight_diego_cell_disk_size: $least_diego_cell_disk_size"
    echo "available_diego_cell_disk_size: $available_diego_cell_disk_size"
    echo $least_diego_cell_disk_size ">" $available_diego_cell_disk_size
    echo "It is necessary to increase the diego_cell disk capacity."
else
    echo "max_in_flight_diego_cell_disk_size: $least_diego_cell_disk_size"
    echo "available_diego_cell_disk_size: $available_diego_cell_disk_size"
    echo $least_diego_cell_disk_size "<" $available_diego_cell_disk_size

fi

if [ $least_diego_cell_memory_size -gt $available_diego_cell_memory_size ]; then
    echo "max_in_flight_diego_cell_memory_size: $least_diego_cell_memory_size"
    echo "available_diego_cell_memory_size: $available_diego_cell_memory_size"
    echo $least_diego_cell_memory_size ">" $available_diego_cell_memory_size
    echo "It is necessary to increase the diego_cell_memory capacity."
else
    echo "max_in_flight_diego_cell_memory_size size: $least_diego_cell_memory_size"
    echo "available_diego_cell_memory_size: $available_diego_cell_memory_size"
    echo $least_diego_cell_memory_size "<" $available_diego_cell_memory_size
fi

echo "########################diego capacity check end########################"

echo "########################bosh clean up start########################"

bosh clean-up --all -n

echo "########################bosh clean up end########################"



```
