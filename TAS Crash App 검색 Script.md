```
DEPLOYMENT="cf-xxxxxxxx"
echo "CRASH APP FIND START"
BOSH_CLIENT=ops_manager BOSH_CLIENT_SECRET=rVdOTQvmXYzd3UgtI0fYsWiwZ-j3N3aj BOSH_CA_CERT=/var/tempest/workspaces/default/root_ca_certificate BOSH_ENVIRONMENT={BOSH_IP} bosh -d  $DEPLOYMENT ssh diego_cell/0 -c \
  "/var/vcap/packages/cfdot/bin/cfdot actual-lrp-groups \
       --bbsURL https://bbs.service.cf.internal:8889 \
       --caCertFile /var/vcap/jobs/cfdot/config/certs/cfdot/ca.crt \
       --clientCertFile /var/vcap/jobs/cfdot/config/certs/cfdot/client.crt \
       --clientKeyFile /var/vcap/jobs/cfdot/config/certs/cfdot/client.key " \
  | awk '($0 ~ /instance/) { $1=$2=$3=""; print $0 }' \
  | jq -r ".instance | \"\(.process_guid[0:36]) \(.index) \(.crash_count) \(.state) \" " \
  | while IFS=" " read -r process_guid index crash_count state
    do 
      if [ 0 -eq $index ]; then
          if [ 0 -ne $crash_count ] && [ $state == "CRASHED" ]; then
            app_state=$(cf curl /v2/apps/$app_guid/stats | jq '."'$j'".state' | tr -d '"')
            crash_app_name=$(cf curl /v2/apps/$process_guid | jq .entity.name | tr -d '"')
            get_space_url=$(cf curl /v2/apps/$process_guid | jq .entity.space_url | tr -d '"')
            get_space_guid=$(cf curl $get_space_url | jq .metadata.guid | tr -d '"')
            get_org_url=$(cf curl /v2/spaces/$get_space_guid | jq .entity.organization_url | tr -d '"')
            crash_app_space=$(cf curl $get_space_url | jq .entity.name | tr -d '"')
            crash_app_org=$(cf curl $get_org_url | jq .entity.name | tr -d '"')  

            echo "APP NAME: $crash_app_name ORG: $crash_app_org SPACE: $crash_app_space"
          fi
      fi
    done
echo "CRASH APP FIND END"
 ```
