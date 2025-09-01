# 자주 묻는 질문

## Terraform Example Foundation을 통해 생성된 프로젝트에서 낮은 할당량이 발생하는 이유는 무엇인가요?

0-bootstrap 단계에서 생성된 서비스 계정으로 Terraform Example Foundation을 배포할 때,
프로젝트 할당량은 사용자 ID가 아닌 서비스 계정의 평판을 기반으로 합니다.
많은 경우 이 할당량은 처음에 낮습니다.

0-bootstrap 단계에서 생성된 서비스 계정 `terraform_service_account`에 대해 50개의 추가 프로젝트를 요청하는 것을 권장합니다.
[프로젝트 할당량 증가 요청](https://support.google.com/code/contact/project_quota_increase) 양식을 사용하여 할당량 증가를 요청할 수 있습니다.
지원 양식에서 **프로젝트 생성에 사용될 이메일 주소**에는 조직 부트스트랩 모듈에서 생성된 `terraform_service_account` 주소를 사용하세요.
다른 할당량 오류가 표시되면 [할당량 문서](https://cloud.google.com/docs/quota)를 참조하세요.

## "named" 브랜치란 무엇인가요?

terraform-example-foundation의 특정 브랜치들은
_named branches_로 간주됩니다. named branch에 푸시하면 _apply_ 명령이
실행됩니다. named branch가 아닌 브랜치에 푸시하면 _apply_가 실행되지 않습니다.

* development
* nonproduction
* production

## 브랜치에 푸시할 때 어떤 Terraform 명령이 실행되나요?

_named branch_에 푸시한 경우 다음 명령들이 실행됩니다: _init_, _plan_, _validate_, _apply_.

named branch가 아닌 브랜치에 푸시하면 _init_, _plan_, _validate_만 실행됩니다.
_apply_ 명령은 실행되지 않습니다.
