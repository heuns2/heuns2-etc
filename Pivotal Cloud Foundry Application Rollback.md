# Pivotal Cloud Foundry Application Rollback

- 작성 문서의 Version 목록은 아래와 같다.
	- OpsManager v2.6.5
	- PAS 2.6.3
- PAS 상에 구동되는 Application에 에러 또는 이상이 발생 했을 때 이전 버전으로 Rollback을 실행 할 수 있는 방법을 기술 하였다.
- 단, cf cli v3을 통해서 가능하며 추후에 다른 버전이 지원 될 예정. 현재는 CAPI v3 (Cloud Controller API v3)을 권장한다.

## 1. Pivotal Cloud Foundry App Revisions
- Application Revision은 Application의 변경 이력, Code 구성을 확인 할 수 있다, CAPI를 통해 Application의 Droplet, Buildpack의 Custom Start Command, Service Environment Variables를 참조 한다.
- App Revisions의 요소
	- Viewing revisions for an app: Application의 주요 변경 사항을 추적
	- Rolling back to a previous revision: 이전 상태를 직접 추적하거나 이전에 실행 한 Application 버전을 배포 할 수 있다.
- 기본적으로 Cloud Foundry는 Application에 대한 100개의 Revision을 가지고 있다.

### 1.1. Events that Trigger Revisions
- Application Revision을 Trigger하는 Event 목록은 아래와 같다.
	- 신규 Droplet이 생성 될 경우
	- Application의 환경 변수가 변경 될 경우
	- Buildpack Custom Start Command가 변경 됬을 경우
	- Application이 이전 버전으로 Rollback 했을 경우

### 1.2. Droplet Storage 고려 사항
- App Revision을 통해 Application Rollback에 필요한 Droplet은 기본으로 2-5개를 가지고 있다, 즉 이전 버전에 대한 Rollback은 최근 Droplet의 2-5개가 가지고 있는 버전만이 가능하다.
- PAS Tile 또는 CF Deploy Manifest 특정 Max Staged Droplets Per App 설정을 변경하여 2~5개 이상으로 갖고 있을 수 있지만 용량에 부하가 발생 할 수 있다.


## 2. Pivotal Cloud Foundry App Revision Test
- App Revision을 사용하여 Test 하기 위해 간단한 Staticfile Buildpack을 사용하여 PAS에 Test라는 Name을 갖은 Application을 Push 
- Revision (Rollback 대상) 버전을 생성하기 위해 Trigger $ cf set-nev test revision test && cf restage test를 실행
- Revision (Rollback 명령 실행) 버전을 생성하기 위해 Trigger $ cf set-nev test revision rollback && cf restage test를 실행

```
$ cf env test
User-Provided: <<< User-Provided에 사용자 제공 환경 변수가 존재하는지 확인
revision: rollback
```
 
### 2.1. Enable Revisions for an App
- Revision 기능을 사용하기 위해 CLI를 사용하여 특정 App의 Revision을 Enable 한다.

```
$ cf app test --guid
4859e183-de2a-4fff-8dfb-96df8d239256

$ cf curl /v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/features/revisions -X PATCH -d '{ "enabled": true }'
{
   "name": "revisions",
   "description": "Enable versioning of an application (experimental)",
   "enabled": true
}
```

### 2.2. List Revisions for an App
- Application의 Revision 목록을 확인 한다.

```
$ cf curl /v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/revisions
{
   "pagination": {
      "total_results": 1,
      "total_pages": 1,
      "first": {
         "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/revisions?page=1&per_page=50"
      },
      "last": {
         "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/revisions?page=1&per_page=50"
      },
      "next": null,
      "previous": null
   },
   "resources": [
      {
         "guid": "6e2af88b-7bf2-4540-aabf-a6d0a90301f1",
         "version": 1,
         "droplet": {
            "guid": "86a41fc4-e979-44c4-9d5d-304529983132"
         },
         "processes": {
            "web": {
               "command": null
            }
         },
         "description": "Initial revision.",
         "relationships": {
            "app": {
               "data": {
                  "guid": "4859e183-de2a-4fff-8dfb-96df8d239256"
               }
            }
         },
         "created_at": "2019-08-30T06:42:32Z",
         "updated_at": "2019-08-30T06:42:32Z",
         "links": {
            "self": {
               "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/revisions/6e2af88b-7bf2-4540-aabf-a6d0a90301f1"
            },
            "app": {
               "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256"
            },
            "environment_variables": {
               "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/revisions/6e2af88b-7bf2-4540-aabf-a6d0a90301f1/environment_variables"
            }
         },
         "metadata": {
            "labels": {},
            "annotations": {}
         }
      }
   ]
}
```

### 2.3. List Deployed Revisions for an App
- 현재 Push 한 Application의 최근 Revision를 확인 할 수 있다.
- 지금은 결과가 같지만 배포 된 Revision 한개만 출력 된다.
```
$ cf curl /v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/revisions/deployed
{
   "pagination": {
      "total_results": 1,
      "total_pages": 1,
      "first": {
         "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/revisions/deployed?page=1&per_page=50"
      },
      "last": {
         "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/revisions/deployed?page=1&per_page=50"
      },
      "next": null,
      "previous": null
   },
   "resources": [
      {
         "guid": "6e2af88b-7bf2-4540-aabf-a6d0a90301f1", ###### revision guid
         "version": 1,
         "droplet": {
            "guid": "86a41fc4-e979-44c4-9d5d-304529983132"
         },
         "processes": {
            "web": {
               "command": null
            }
         },
         "description": "Initial revision.",
         "relationships": {
            "app": {
               "data": {
                  "guid": "4859e183-de2a-4fff-8dfb-96df8d239256"
               }
            }
         },
         "created_at": "2019-08-30T06:42:32Z",
         "updated_at": "2019-08-30T06:42:32Z",
         "links": {
            "self": {
               "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/revisions/6e2af88b-7bf2-4540-aabf-a6d0a90301f1"
            },
            "app": {
               "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256"
            },
            "environment_variables": {
               "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/revisions/6e2af88b-7bf2-4540-aabf-a6d0a90301f1/environment_variables"
            }
         },
         "metadata": {
            "labels": {},
            "annotations": {}
         }
      }
   ]
}
```

### 2.4. Rollback to a Previous Revision
- 이전 Revision으로 Rollback 할 수 있다.

```
$ cf curl v3/deployments \
-X POST \
-d '{
  "revision": {
    "guid": "6e2af88b-7bf2-4540-aabf-a6d0a90301f1"
  },
  "relationships": {
    "app": {
      "data": {
        "guid": "4859e183-de2a-4fff-8dfb-96df8d239256"
      }
    }
  }
}'



{
   "guid": "1477334b-b3cd-4780-9e49-c27318d04cf9",
   "state": "DEPLOYING",
   "droplet": {
      "guid": "86a41fc4-e979-44c4-9d5d-304529983132"
   },
   "previous_droplet": {
      "guid": "86a41fc4-e979-44c4-9d5d-304529983132"
   },
   "new_processes": [
      {
         "guid": "6448a60a-9aa3-4f1f-9a5a-b3e847dc1238",
         "type": "web"
      }
   ],
   "created_at": "2019-08-30T07:21:22Z",
   "updated_at": "2019-08-30T07:21:22Z",
   "relationships": {
      "app": {
         "data": {
            "guid": "4859e183-de2a-4fff-8dfb-96df8d239256"
         }
      }
   },
   "metadata": {
      "labels": {},
      "annotations": {}
   },
   "links": {
      "self": {
         "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/deployments/1477334b-b3cd-4780-9e49-c27318d04cf9"
      },
      "app": {
         "href": "https://api.sys.{SYSTEM_DOMAIN}/v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256"
      }
   },
   "revision": {
      "guid": "9df3ebe1-0c18-4f95-bdcb-b41addc317fb",
      "version": 2
   }
}
```

### 2.5. "2.4"의 Rollback 실행 결과 결과 확인

-  Rollback 실행 전 환경 변수 설정 값은 revision: rollback

```
$ cf env test
User-Provided:
revision: rollback
```

-  Rollback 실행 후 환경 변수 설정 값이 처음 설정한 revision: test로 변경 됨을 확인

```
$ cf env test
User-Provided:
revision: test
```

- $ cf curl /v3/apps/4859e183-de2a-4fff-8dfb-96df8d239256/revision를 다시 실행하여 최신의 revision guid를 통해 다시 아래 명령어 실행

```
$ cf curl v3/deployments -X POST -d '{
  "revision": {
    "guid": "d9fd3020-cb60-4d46-89ff-cc63c21e3fb8"
  },
  "relationships": {
    "app": {
      "data": {
        "guid": "4859e183-de2a-4fff-8dfb-96df8d239256"
      }
    }
  }
}'
``` 
-  Rollback 실행 후 환경 변수 설정 값이 다시 최신의 revision: rollback으로 변경되었는지 확인
```
$ cf env test
User-Provided:
revision: rollback
```

- Revision의 변경 사항 확인
```
cf curl /v3/apps/"4859e183-de2a-4fff-8dfb-96df8d239256"/revisions | grep description
         "description": "Initial revision.",
         "description": "Rolled back to revision 1.",
         "description": "Rolled back to revision 1.",
         "description": "New droplet deployed. New environment variables deployed.",
         "description": "Rolled back to revision 1.",
         "description": "Rolled back to revision 4.",
```
