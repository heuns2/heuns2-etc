
```
#!/bin/bash

export timestamp="$(date '+%Y%m%d.%-H%M.%S+%Z')"

mv ./jq-binary/jq* ./jq
mv ./cf-binary/cf* ./cf
chmod +x ./jq ./cf



admin_id=$(om --env ./env/$ENV_FILE credentials --product-name cf --credential-reference .uaa.admin_credentials -t json | ./jq .identity | tr -d '"')
admin_pwd=$(om --env ./env/$ENV_FILE credentials --product-name cf --credential-reference .uaa.admin_credentials -t json | ./jq .password | tr -d '"')

./cf login -a https://api.sys.{DOMA} -u $admin_id -p $admin_pwd -o $FOUNDATION -s common --skip-ssl-validation


function vm_status_check(){
  if [[ $1 -eq 100 ]]; then
    curl -G "https://log-cache.sys.{DOMAIN}/api/v1/query" --data-urlencode 'query=system_healthy{source_id="bosh-system-metrics-forwarder"} == 0' \
    -H "Authorization: $(./cf oauth-token)" | \
    ./jq '[ .data.result[] | {
    deployment_name:.metric.deployment,
    job_name:.metric.job,
    index:.metric.index,
    status: .value[1],
    date: .value[0]
    }]' > vm_staus_check.json
    index=$(cat vm_staus_check.json | ./jq length | tr -d '"')
  fi
  
  if [[ $1 -eq 200 ]]; then
    cp vm_staus_check.json vm_staus_check_old.json
    sleep 5;
    curl -G "https://log-cache.sys.{DOMAIN}/api/v1/query" --data-urlencode 'query=system_healthy{source_id="bosh-system-metrics-forwarder"} == 0' \
    -H "Authorization: $(./cf oauth-token)" | \
    ./jq '[ .data.result[] | {
    deployment_name:.metric.deployment,
    job_name:.metric.job,
    index:.metric.index,
    status: .value[1],
    date: .value[0]
    }]' > vm_staus_check.json
    sleep 5;
    index=$(cat vm_staus_check.json | ./jq length | tr -d '"')
  fi
  sleep 5;
  if [[ $1 -eq 200 ]]; then
    old_index=$(cat vm_staus_check_old.json | ./jq length | tr -d '"')
    for ((i=0;i<$old_index;i++))
    do
      old_deployment_name=$(cat vm_staus_check_old.json | ./jq .[$i].deployment_name | tr -d '"')
      old_job_name=$(cat vm_staus_check_old.json | ./jq .[$i].job_name | tr -d '"')
      old_index_name=$(cat vm_staus_check_old.json | ./jq .[$i].index | tr -d '"')
      grep_new_data="$(cat vm_staus_check.json | ./jq . | grep $old_index_name)"
      echo $grep_new_data
      if [[ $grep_new_data == "" ]]; then
        echo $FOUNDATION $old_job_name/$old_index_name "VM UP"
      fi
    done
  fi
  vm_status_result $index $1
}

function vm_status_result(){
  sleep 5;
  if [ $1 -eq  0 ] && [ $2 -eq  100 ]; then
    echo "VM Status All Done"
  else
    for ((i=0;i<$1;i++))
    do
      deployment_name=$(cat vm_staus_check.json | ./jq .[$i].deployment_name | tr -d '"')
      job_name=$(cat vm_staus_check.json | ./jq .[$i].job_name | tr -d '"')
      index_name=$(cat vm_staus_check.json | ./jq .[$i].index | tr -d '"')
      
    done
    if [ $1 -eq  0 ]; then
      echo "VM Status All Done"
      exit 0
    fi

    while true; do
        vm_status_check 200
    done
  fi
}

vm_status_check 100

```
