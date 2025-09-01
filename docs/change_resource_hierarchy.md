# 리소스 계층 구조 사용자 지정

이 문서는 Terraform Foundation Example 블루프린트 배포 중 Cloud Resource Manager 리소스 계층 구조, 폴더 및 프로젝트를 사용자 지정하기 위한 가이드를 제공합니다.

Terraform Foundation Example 블루프린트의 현재 배포 시나리오는 모든 폴더가 동일한 수준에 있고 각 환경마다 하나의 폴더와 세 개의 특별 폴더를 갖는 평면적 리소스 계층 구조를 고려합니다. 각 폴더에 대한 자세한 설명은 다음과 같습니다:

| 폴더 | 설명 |
| --- | --- |
| bootstrap | 파운데이션 구성 요소를 배포하는 데 사용되는 시드 및 CI/CD 프로젝트를 포함합니다. |
| common | 로깅 및 Security Command Center와 같이 조직에서 사용하는 공통 리소스가 있는 프로젝트를 포함합니다. |
| network | DNS Hub, 하이브리드 연결, Shared VPC와 같이 조직에서 사용하는 공통 네트워크 리소스가 있는 프로젝트를 포함합니다. |
| production | 프로덕션으로 승격된 클라우드 리소스가 있는 프로젝트를 포함하는 환경 폴더입니다. |
| nonproduction | 프로덕션에 투입하기 전에 워크로드를 테스트할 수 있도록 프로덕션 환경의 복제본을 포함하는 환경 폴더입니다. |
| development | 개발 및 샌드박스 환경으로 사용되는 환경 폴더입니다. |

이 문서는 소스 코드 관점과 Cloud Resource Manager 관점 모두에서 환경 중심적 초점을 가지고 두 개 이상의 폴더 수준을 가질 수 있는 시나리오를 다룹니다: `환경 -> ... -> 비즈니스 유닛`.

| 현재 계층 구조 | 변경된 계층 구조 |
| --- | --- |
| <pre>example-organization/<br>├── fldr-bootstrap<br>├── fldr-common<br>├── fldr-network<br>├── <b>fldr-development *</b><br>├── <b>fldr-nonproduction *</b><br>└── <b>fldr-production *</b><br></pre> | <pre>example-organization/<br>├── fldr-bootstrap<br>├── fldr-common<br>├── fldr-network<br>├── <b>fldr-development *</b><br>│   ├── finance<br>│   └── retail<br>├── <b>fldr-nonproduction *</b><br>│   ├── finance<br>│   └── retail<br>└── <b>fldr-production *</b><br>    ├── finance<br>    └── retail<br></pre> |

## 코드 변경 - 빌드 파일

`tf-wrapper.sh` 파일은 Terraform Foundation Example 블루프린트에 대한 Terraform 구성을 적용하는 데 책임이 있는 bash 스크립트 헬퍼입니다.
`tf-wrapper.sh` 스크립트는 Terraform Example Foundation에 정의된 [브랜치 전략](../README.md#branching-strategy)을 기반으로 작동합니다.
이 스크립트는 현재 git 브랜치 이름과 일치하는 폴더 이름(환경을 리프 노드로 사용)을 찾기 위해 소스 코드 폴더 계층 구조를 스캔하고 해당 폴더에 포함된 terraform 구성을 적용합니다.

다음 변경 사항은 `tf-wrapper.sh` 스크립트가 매칭 폴더를 더 깊이 검색하고 이 문서에서 제시된 소스 코드 폴더 계층 구조를 준수할 수 있도록 구성합니다.

참고: 현재 브랜치 이름이 소스 코드 폴더 계층 구조의 루트에 있고 비즈니스 유닛이 리프가 되도록(환경을 루트 노드로 사용) 소스 코드를 스캔하도록 `tf-wrapper.sh` 스크립트를 구성하는 것도 가능합니다. 이 대안에 대한 자세한 내용은 `tf-wrapper.sh` 스크립트를 참조하세요.

1. `tf-wrapper.sh` 스크립트에서 `max_depth` 변수 값을 `2`로 설정합니다.

```bash
max_depth=2
```

## 코드 변경 - Terraform 파일

<pre>
example-organization/
├── bootstrap
├── common
├── <b>development *</b>
│   ├── finance
│   └── retail
├── <b>nonproduction *</b>
│   ├── finance
│   └── retail
└── <b>production *</b>
    ├── finance
    └── retail
</pre>

*예제 1 - 계층 구조가 변경된 Terraform Foundation Example의 예시*

### 2-environments 단계

1. `env_baseline` 모듈에서 비즈니스 유닛용 폴더 계층 구조를 생성하여 모든 환경에서 동일하게 복제되도록 합니다.

    예시:

    2-environments/modules/env_baseline/folders.tf

    ```text
    ...
    /******************************************
        환경 폴더
    *****************************************/

    resource "google_folder" "env" {
        display_name = "${local.folder_prefix}-${var.env}"
        parent       = local.parent
    }

    /* 🟢 폴더 계층 구조 생성 */
    resource "google_folder" "finance" {
        display_name = "finance"
        parent       = google_folder.env.name
    }

    resource "google_folder" "retail" {
        display_name = "retail"
        parent       = google_folder.env.name
    }
    ...
    ```

1. `env_baseline` 모듈에서 새로운 계층 구조의 평면적 표현을 가진 출력을 생성합니다.

    *표 1 - 예제 1 리소스 계층 구조에 대한 출력 예시*

    | 폴더 경로 | 폴더 ID |
    | --- | --- |
    | development | folders/0000000 |
    | development/finance | folders/11111111 |
    | development/retail | folders/2222222 |

    *표 2 - 더 많은 수준을 가진 리소스 계층 구조에 대한 출력 예시*

    | 폴더 경로 | 폴더 ID |
    | --- | --- |
    | development | folders/0000000 |
    | development/us | folders/11111111 |
    | development/us/finance | folders/2222222 |
    | development/us/retail | folders/3333333 |
    | development/europe | folders/4444444 |
    | development/europe/finance | folders/5555555 |
    | development/europe/retail | folders/7777777 |

    예시:

    2-environments/modules/env_baseline/outputs.tf

    ```text
    ...
    /* 🟢 폴더 계층 구조 출력 */
    output "folder_hierarchy" {
        description = "프로젝트가 생성되어야 하는 새로운 폴더 계층 구조의 평면적 표현을 가진 맵입니다."
        value       = {
        "${google_folder.env.display_name}" = google_folder.env.name
        "${google_folder.env.display_name}/finance" = google_folder.finance.name
        "${google_folder.env.display_name}/retail" = google_folder.retail.name
        }
    }
    ```

1. 각 환경에서 `env_baseline` 모듈의 새로운 계층 구조를 평면적으로 표현한 출력을 생성합니다. 이는 다음 단계에서 GCP 프로젝트를 호스팅하는 데 사용됩니다.

    예시:

    2-environments/envs/development/outputs.tf

    ```text
    ...
    /* 🟢 폴더 계층 구조 출력 */
    output "folder_hierarchy" {
        description = "프로젝트가 생성되어야 하는 새로운 폴더 계층 구조의 평면적 표현을 가진 맵입니다."
        value       = module.env.folder_hierarchy
    }
    ```

1. 배포를 진행합니다.

### 4-projects 단계

1. base_env 모듈이 2-environments 단계의 계층 구조 맵에서 새로운 폴더 키(예: development/retail)를 받도록 변경합니다.
1. 이 폴더 키는 프로젝트가 생성되어야 하는 폴더를 가져오는 데 사용되어야 합니다.
    예시:

    4-projects/modules/base_env/variables.tf

    ```text
    ...
    /* 🟢 폴더 키 변수 */
    variable "folder_hierarchy_key" {
        description = "프로젝트가 생성되어야 하는 폴더를 가져오기 위한 폴더 계층 구조 맵의 키입니다."
        type = string
        default = ""
    }
    ...
    ```

    4-projects/modules/base_env/main.tf

    ```text
    locals {
        ...
        /* 🟢 새로운 폴더 가져오기 */
        env_folder_name = lookup(
        data.terraform_remote_state.environments_env.outputs.folder_hierarchy, var.folder_hierarchy_key
        , data.terraform_remote_state.environments_env.outputs.env_folder)
        ...
    }
    ...
    ```

1. 환경 폴더(development, nonproduction, production) 위에 소스 코드 폴더 계층 구조를 생성합니다. 소스 코드 폴더 계층 구조에서 소스 코드 환경 폴더를 리프(최고 수준)로 유지해야 한다는 점을 기억하세요. 이는 terraform 구성을 적용하기 위해 bash 스크립트 헬퍼인 `tf-wrapper.sh`가 작동하는 방식이기 때문입니다.
1. 필요에 맞게 소스 폴더 계층 구조를 수동으로 복제합니다.
1. **(์„ ํƒ์‚ฌํ•ญ)** 아래 변경 사항에서 business_units 이름 변경을 단순화하기 위해 도움이 되는 스크립트를 제공합니다. **변경 사항을 반드시 검토하세요**. 아래 스크립트는 `gcp-projects` 폴더에 있다고 가정합니다:

    ```bash
    for i in `find "./business_unit_1" -type f -not -path "*/.terraform/*" -name '*.tf'`; do sed -i'' -e "s/bu1/<귀하의 비즈니스 유닛 코드>/" $i; done

    for i in `find "./business_unit_1" -type f -not -path "*/.terraform/*" -name '*.tf'`; do sed -i'' -e "s/business_unit_1/<귀하의 비즈니스 유닛 이름>/" $i; done

    for i in `find "./business_unit_2" -type f -not -path "*/.terraform/*" -name '*.tf'`; do sed -i'' -e "s/bu2/<귀하의 비즈니스 유닛 코드>/" $i; done

    for i in `find "./business_unit_2" -type f -not -path "*/.terraform/*" -name '*.tf'`; do sed -i'' -e "s/business_unit_2/<귀하의 비즈니스 유닛 이름>/" $i; done

    for i in `find "./business_unit_<새로운 비즈니스 유닛 번호>" -type f -not -path "*/.terraform/*" -name '*.tf'`; do sed -i'' -e "s/bu<새로운 비즈니스 유닛 번호>/<귀하의 비즈니스 유닛 코드>/" $i; done

    for i in `find "./business_unit_<새로운 비즈니스 유닛 번호>" -type f -not -path "*/.terraform/*" -name '*.tf'`; do sed -i'' -e "s/business_unit_<새로운 비즈니스 유닛 번호>/<귀하의 비즈니스 유닛 이름>/" $i; done
    ```

1. 이 예시에서는 예시 폴더 계층 구조와 일치하도록 business_unit_1과 business_unit_2 폴더를 비즈니스 유닛 이름(์˜ˆ: finance 및 retail)으로 이름을 변경하기만 하면 됩니다.





1. 각 비즈니스 유닛 공유 리소스에 대해 backend gcs prefix를 변경합니다.
    예시:

    4-projects/finance/shared/backend.tf

    ```text
    ...
    terraform {
        backend "gcs" {
            bucket = "<YOUR_PROJECTS_BACKEND_STATE_BUCKET>"

            /* 🟢 prefix 경로 검토 */
            prefix = "terraform/projects/finance/shared"
        }
    }
    ```

1. Cloud Build 프로젝트 파이프라인에서 로컬 `repo_names` 값을 검토합니다. 이 이름은 `4-projects/modules/base_env/example_base_shared_vpc_project.tf`에서 base_shared_vpc_project 모듈 변수의 `sa_roles` 키와 일치해야 합니다. 이 값의 현재 패턴은 `"${var.business_code}-example-app"`입니다.
1. Cloud Build 프로젝트 파이프라인에서 비즈니스 코드를 검토합니다.
    예시:

    4-projects/finance/shared/example_infra_pipeline.tf

    ```text
    locals {
        /* 🟢 로컬 변수 검토 */
        repo_names = ["fin-example-app"]
    }
    ...

    module "app_infra_cloudbuild_project" {

        /* 🟢 모듈 경로 검토 */
        source = "../../modules/single_project"
        ...
        primary_contact   = "example@example.com"
        secondary_contact = "example2@example.com"

        /* 🟢 비즈니스 코드 검토 */
        business_code     = "fin"
    }
    ```

1. 각 비즈니스 유닛 환경에 대해 backend gcs prefix를 변경합니다.
    예시:

    4-projects/finance/development/backend.tf

    ```text
    ...
    terraform {
        backend "gcs" {
            bucket = "<YOUR_PROJECTS_BACKEND_STATE_BUCKET>"

            /* 🟢 prefix 경로 검토 */
            prefix = "terraform/projects/finance/development"
        }
    }
    ```

1. 새로운 비즈니스 유닛 이름과 일치하도록 business_code와 business_unit을 검토합니다.
1. base_env 호출에서 새로운 folder_hierarchy_key 매개변수를 설정합니다.

    예시:

    4-projects/finance/development/main.tf

    ```text
    module "env" {
        /* 🟢 모듈 경로 검토 */
        source = "../../modules/base_env"

        env                  = "development"

        /* 🟢 비즈니스 코드 검토 */
        business_code        = "fin"
        business_unit        = "finance"

        /* 🟢 폴더 키 매개변수 설정 */
        folder_hierarchy_key = "fldr-development/finance"
        ...
    }
    ```

1. 배포를 진행합니다.
