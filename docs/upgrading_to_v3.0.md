# 업그레이드 가이드
v3 구성 요소 채택을 진행하기 전에 아래의 주요 변경 사항 목록을 검토하시기 바랍니다. 기능, 버그 수정 및 기타 업데이트의 전체 목록은 [변경 로그](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/CHANGELOG.md)에서 확인할 수 있습니다.

**중요:** v2에서 v3로의 현재 위치 업그레이드 경로는 제공되지 않습니다.

## 주요 변경 사항

- 최소 필수 Terraform 버전이 이제 1.3.0입니다. 이전 릴리스에서는 최소 버전이 0.13.7이었습니다.
- [BYOSA 기능](https://cloud.google.com/build/docs/securing-builds/configure-user-specified-service-accounts)을 사용하여 Cloud Build 내에서 활용되는 각 단계별 세분화된 서비스 계정(SA)이 추가되었습니다. 이전 버전에서는 모든 단계를 배포하기 위해 단일 SA가 사용되어 과도한 권한이 부여되었습니다. 이제 각 단계는 매우 제한된 권한을 가진 자체 SA를 보유합니다.
- 3-networks 단계가 두 개의 서로 다른 디렉토리로 분할되었습니다. 이전에는 3-networks 단계가 Dual Shared VPC와 Hub and Spoke 두 가지 네트워크 모드를 모두 지원했습니다. 이번 릴리스에서는 더 쉬운 사용자 지정 및 유지 관리를 위해 이 두 모드가 두 개의 서로 다른 구현으로 분리되었습니다.

**참고:** 이전 버전을 사용하는 이미 배포된 구성 위에서 직접 새로운 Terraform 버전을 사용할 수 있습니다. 그러나 이렇게 하면 이전 버전을 사용할 수 없게 됩니다. 또한 이는 terraform 상태와 종속성(외부 모듈 및 제공자의 리소스)을 업데이트합니다. 따라서 코드베이스를 업그레이드하기 전에 현재 terraform 상태를 백업하는 것을 권장합니다.

## 새로운 기능 통합

v2에서 v3로의 직접적인 업그레이드 경로는 없습니다. 이는 리소스가 삭제되고 재생성될 수 있기 때문입니다.

v3의 일부 기능을 통합해야 하는 경우, 관심 있는 기능에 대한 문서를 검토하고 v3의 코드를 구현 가이드로 사용하는 것을 권장합니다. 또한 업데이트를 적용하기 전에 파괴적인 작업에 대한 `terraform plan` 출력을 검토하는 것을 권장합니다.

**참고:** `terraform`과 `gcloud`에 대해 올바른 버전을 사용하고 있는지 확인해야 합니다. 이 [검증 스크립트](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/scripts/validate-requirements.sh)를 사용하여 이러한 요구사항과 기타 추가 요구사항을 확인할 수 있습니다.

### 이동 블록

코드베이스에 기능을 통합하면 일부 리소스가 상위 모듈에서 하위 모듈로, 하위 모듈에서 상위 모듈로, 또는 외부 모듈에서 구성으로 이동될 수 있습니다.

이러한 다양한 시나리오를 고려하여 리소스를 업데이트하고 코드를 안전하게 리팩터링할 수 있게 해주는 `moved` 블록을 사용하는 것을 제안합니다. 자세한 내용은 [이동 블록](https://developer.hashicorp.com/terraform/tutorials/configuration-language/move-config)을 참조하세요.

**참고:** `moved` 블록은 예제 파운데이션 v3에 필요한 terraform 버전(v1.3.0)에서 지원됩니다.

다음으로, 이러한 이동 블록이 어떻게 구현될 수 있는지에 대한 몇 가지 예제를 제공합니다.

### 모듈 간 이동

수정 없이 배포된 0-bootstrap 파운데이션 v2를 고려해보겠습니다. v3로 마이그레이션을 시도하면 `terraform plan` 출력의 요약으로 다음과 같은 내용을 관찰할 수 있습니다.

```hcl
    Plan: 165 to add, 2 to change, 67 to destroy.
```

`module.cloudbuild_bootstrap.module.cloudbuild_project` 모듈을 검토하면 이와 관련된 많은 리소스가 다른 모듈에서 삭제되고 다시 생성되는 것을 관찰할 수 있습니다.

```hcl
    # module.cloudbuild_bootstrap.module.cloudbuild_project.module.project-factory.module.project_services.google_project_service.project_services["admin.googleapis.com"] will be destroyed
    # (because google_project_service.project_services is not in configuration)
    - resource "google_project_service" "project_services" {
      - disable_dependent_services = true -> null
      - disable_on_destroy         = false -> null
      - id                         = "prj-b-cicd-9aee/admin.googleapis.com" -> null
      - project                    = "prj-b-cicd-9aee" -> null
      - service                    = "admin.googleapis.com" -> null
    }
```

And

```hcl
    # module.tf_source.module.cloudbuild_project.module.project-factory.module.project_services.google_project_service.project_services["admin.googleapis.com"] will be created
    + resource "google_project_service" "project_services" {
      + disable_dependent_services = true
      + disable_on_destroy         = false
      + id                         = (known after apply)
      + project                    = (known after apply)
      + service                    = "admin.googleapis.com"
    }
```

이 경우, 다음 코드 스니펫을 제공하여 Terraform이 이 리소스 주소를 안전하게 이동하도록 지시할 수 있습니다.

```hcl
    # cloudbuild bootstrap
    moved {
        from = module.cloudbuild_bootstrap.module.cloudbuild_project
        to   = module.tf_source.module.cloudbuild_project
    }
```

**참고:** 이 코드는 별도의 *terraform 파일* (예: moved.tf)에서 구현할 수 있습니다.

대부분의 경우 이것으로 terraform 구성을 업데이트하기에 충분합니다. 그러나 이 특정 예제에서는 소스 모듈의 ID가 자동 생성된 임의의 문자열로 구성되므로 업데이트된 리소스에서도 이 모듈의 ID를 하드코딩해야 합니다.

```hcl
    module "tf_source" {
        source  = "terraform-google-modules/bootstrap/google//modules/tf_cloudbuild_source"
        version = "~> 6.2"

        org_id                = var.org_id
        folder_id             = google_folder.bootstrap.id
        project_id            = "${var.project_prefix}-b-cicd-${random_string.suffix.result}" # REPLACE_HERE
        billing_account       = var.billing_account
        group_org_admins      = local.group_org_admins
        buckets_force_destroy = var.bucket_force_destroy
    ...
    }
```

삭제될 리소스 수가 줄어든 것을 확인할 수 있습니다.

    Plan: 146 to add, 4 to change, 48 to destroy.

### 백업

`moved` 블록을 사용하여 리소스가 삭제되는 것을 방지하고 대신 백업으로 복사본을 만들 수도 있습니다.

다음 코드 스니펫은 파운데이션 v3 구성의 일부가 더 이상 아닌 KMS 키를 저장하는 방법을 보여줍니다.

```hcl
terraform-example foundation/0-bootstrap/backup.tf.example

resource "google_kms_crypto_key" "backup_tf_key" {
    destroy_scheduled_duration = "86400s"
    import_only                   = false
    key_ring                      = "projects/<PROJECT_ID>/locations/us-central1/keyRings/tf-keyring"
    labels                        = {}
    name                          = "tf-key"
    purpose                       = "ENCRYPT_DECRYPT"
    skip_initial_version_creation = false

    version_template {
        algorithm        = "GOOGLE_SYMMETRIC_ENCRYPTION"
        protection_level = "SOFTWARE"
    }
}

resource "google_kms_key_ring" "backup_tf_keyring" {
    location = "us-central1"
    name     = "tf-keyring"
    project  = <PROJECT_ID>
}
```

```hcl
terraform-example foundation/0-bootstrap/moved.tf.example

moved {
    from = module.cloudbuild_bootstrap.google_kms_crypto_key.tf_key
    to   = google_kms_crypto_key.backup_tf_key
}

moved {
    from = module.cloudbuild_bootstrap.google_kms_key_ring.tf_keyring
    to   = google_kms_key_ring.backup_tf_keyring
}
```
